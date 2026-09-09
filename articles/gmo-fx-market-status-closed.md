---
title: "GMOコインFX APIで閉場なのに建玉取得が通った話 — 参照系では開閉が分からない"
emoji: "🕒"
type: "tech"
topics: ["gmocoin", "fx", "python", "api"]
published: true
---

## TL;DR

- 閉場中(土曜)でも GMOコインFX の Private 参照系 GET は **HTTP 200 + ボディ `status: 0`** を返します。建玉一覧も注文一覧も残高も、通常どおり応答しました
- そのため「参照系 GET が通る = 取引時間内」という代理判定は成立しません。閉場を検知できずに素通りします
- 正しい判定は `GET /public/v1/status` の **`data.status`** です。値は `MAINTENANCE` / `CLOSE` / `OPEN`(2026年8月時点の公式)で、閉場中の実測は `CLOSE` でした
- ボディ外側の `status`(API 成否の 0/1)と `data.status`(市場状態)は**同名で別物**です。取り違えると「成功したから開いている」と読んでしまいます

## 背景

GMOコインFX の API を使う自動売買システム(AWS Lambda + Python)を作っています。全建玉を決済するシーケンス(定時の手仕舞いと、緊急時の非常停止で共有)の先頭には、**市場が開いているかを確認する段(以下 Step0)** を置いています。閉場中に走っても発注系は通らないので、その手前で止めるための段です。

この Step0 を、当初は「**参照系の Private GET が正常に返れば取引時間内**」という代理判定で実装していました。建玉一覧が取れないなら何もできないのだから、取れること自体を通過条件にすればよい、という発想です。

ただ、この判定が本当に閉場を弾けるのかは確かめていませんでした。そこで FX が閉まっている土曜に、**GET だけ**(発注も取消もしない読み取りのみ)で実レスポンスを採りました。

## 問題

閉場中でも、Private の参照系 GET は**すべて HTTP 200 + ボディ `status: 0`** を返しました。

| エンドポイント | 閉場中の実レスポンス |
|---|---|
| `GET /private/v1/openPositions` | `HTTP 200` / `{"status": 0, "data": {"list": []}}` |
| `GET /private/v1/activeOrders` | `HTTP 200` / `{"status": 0, "data": {"list": []}}` |
| `GET /private/v1/account/assets` | `HTTP 200` / `{"status": 0, "data": {...}}`(残高も通常どおり) |

エラーにならないどころか、`status: 0`(正常終了)です。なお `list` が空なのは実測時の口座に建玉も注文もなかったためで、閉場の帰結ではありません。中身が空でも埋まっていても、**成否だけでは開閉は分かりません**。

つまり代理判定から見ると、**閉場中も「取引時間内」と読める応答しか返ってきません**。Step0 は素通りします。

## 原因

GMOコインFX の API は、業務エラーも HTTP 200 で返し、成否はボディの `status` で表します(この仕様自体でハマった話は[HTTP 200 なのに約定が0件になった記事](https://zenn.dev/ozapon/articles/gmo-fx-http200-empty-executions)に書きました)。

ここで重要なのは、**参照系 GET の成否が表しているのは「API が正常に応答したか」だけ**という点です。市場が開いているかどうかとは独立した情報でした。閉場中でも口座の状態は参照できるので、参照系が成功するのはむしろ当然の挙動です。

代理判定が検知できるのは、実際には「API が壊れている」ことだけでした。閉場という状態は、参照系 GET の応答のどこにも現れません。自分は「参照が通る」を「取引できる」と読み替えていましたが、**参照が通ることと、取引できることは別**でした。

Step0 が素通りすると、シーケンスはそのまま取消・決済という**発注系 POST に進みます**。実測は GET のみなので閉場中の POST 応答は採っていませんが、少なくとも「閉場エラーで失敗し得る経路に入る」ことは避けたい状態です。

## 解決策

主判定を、公式の市場ステータスに置き換えます。

### `GET /public/v1/status` の `data.status` を見る

公式ドキュメントでは、外国為替FXステータスとして `MAINTENANCE` / `CLOSE` / `OPEN` の3値が定義されています(2026年8月時点)。閉場中の実レスポンスは次のとおりでした。

```json
{"status": 0, "data": {"status": "CLOSE"}, "responsetime": "2026-08-08T04:42:49.180Z"}
```

このレスポンスに、冒頭で触れた同名フィールドの対比がそのまま出ています。外側の `status: 0` は「API 呼び出しが成功した」、`data.status: "CLOSE"` は「市場は閉まっている」です。**外側だけを見ると成功なので、うっかり開場と読める**形になっています。

- **Public API なので署名不要**です。認証まわりの準備なしに呼べます
- レートリミットは GET 枠です。POST の 1回/秒とは別枠ですが、クライアント側の間隔制御は Public GET にも同じように適用しています(この制御の設計は[レートリミットの記事](https://zenn.dev/ozapon/articles/gmo-fx-rate-limiter)に書きました)

### 判定は「OPEN のときだけ続行」に倒す

実装では、`OPEN` に**完全一致したときだけ**続行します。`CLOSE` / `MAINTENANCE` はもちろん、未知の値・空・取得自体の失敗も、すべて閉場扱い(fail-safe)です。

そのうえで、参照系 GET の健全性チェックは**残します**。閉場の検知には使えませんが、「市場は開いているが API 側が壊れている」ケースの検知としては有効だからです。**2つを両方満たしたときだけ**続行します。

```python
MARKET_STATUS_OPEN = "OPEN"


def is_market_open(client, symbol: str) -> bool:
    # 1) 公式の市場ステータス(GET /public/v1/status の data.status)
    try:
        market_status = client.get_market_status()
    except Exception:
        return False  # ステータス不明は閉場扱い(fail-safe)

    if market_status != MARKET_STATUS_OPEN:
        return False  # CLOSE / MAINTENANCE / 未知の値

    # 2) 参照系 GET の健全性(市場は開いているが API が壊れているケースの検知)
    try:
        resp = client.private_get(OPEN_POSITIONS, params={"symbol": symbol})
        raise_for_gmo_status(resp, context="openPositions")  # ボディ status を検査
        return True
    except Exception:
        return False
```

閉場・ステータス不明で止めたときは、処理を進めず**次回起動に持ち越し**ます。非常停止から呼ばれた場合も、応答を成功(`ok`)には倒しません。「止められなかった」ことが成功として記録されると、誰も再試行しないためです(この「成功と誤認しない」設計方針は[非常停止(Kill Switch)の記事](https://zenn.dev/ozapon/articles/gmo-fx-kill-switch-idempotency)にまとめています)。

### 補足: ticker の価格も「開場の証拠」にはならない

`GET /public/v1/ticker` のレスポンスは、**シンボルごとに `status` を持ちます**。そして閉場中でも直近価格自体は返ってきます。

```json
{"status": 0, "data": [
  {"symbol": "USD_JPY", "ask": "157.84", "bid": "157.74",
   "timestamp": "2026-08-08T04:42:49.104119Z", "status": "CLOSE"}, ...]}
```

価格が入っているので、値の有無だけを見る実装は素通りします。**「ticker から価格が返る = 開場」も成り立たない**、という点だけ押さえておけば十分です。価格を使う側でも、必要なら同じ `status` を見ることになります。

## まとめ

開閉確認を参照系 GET の成否で代用すると、閉場を素通りします。閉場の検知と API 障害の検知は別の目的なので、片方でもう片方を兼ねさせません。

手仕舞いや非常停止の先頭に置くなら、公式ステータスを主判定にします。参照系の健全性チェックは残してよいですが、開場の証拠にはしません。

同じシステムの非常停止の冪等設計は[非常停止(Kill Switch)の記事](https://zenn.dev/ozapon/articles/gmo-fx-kill-switch-idempotency)、HTTP 200 の業務エラーを空配列として握りつぶした話は[約定0件に化けた記事](https://zenn.dev/ozapon/articles/gmo-fx-http200-empty-executions)、決済の前に取消の反映を待つ話は[決済前キャンセルの記事](https://zenn.dev/ozapon/articles/gmo-fx-err423-ordered-size)、レートリミットの設計は[レートリミットの記事](https://zenn.dev/ozapon/articles/gmo-fx-rate-limiter)に書いています。

閉場中の ticker が HTTP 200 のまま価格を返し、エンベロープの `status` だけでは開閉が分からない、という穴の型は、実発注しない執行観測デモでも同じでした。[実発注しないデモで執行を観測する記事](https://zenn.dev/ozapon/articles/gmo-fx-virtual-broker)に書いています。
