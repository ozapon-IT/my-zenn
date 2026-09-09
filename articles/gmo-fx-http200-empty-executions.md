---
title: "GMOコインFX APIでHTTP 200なのに約定が0件になった話 — ボディのstatusを見ない罠"
emoji: "📭"
type: "tech"
topics: ["gmocoin", "fx", "python", "api"]
published: true
---

## TL;DR

- GMOコインFX の Private API は、業務エラーでも **HTTP 200** + ボディ `{"status": 1, "messages": [...]}` を返します
- `httpx.Response.raise_for_status()` だけでは通過します。`data` が無いレスポンスを素朴にパースすると **空配列を返して正常終了**し、API エラーが「約定なし」に化けます
- 実際には約定・決済が完了していたのに、取得結果が 0 件扱いになり、下流の日次ノートが**事実と食い違う文面**を出したことがあります
- 対策は約定系 GET でも **ボディの `status` を必ず検査**すること(`raise_for_gmo_status`)。一過性の失敗は同一 invocation 内で短くリトライします
- 認証まわりで同じ「HTTP 200 + ボディ status」を踏んだ話は [ERR-5010の記事](https://zenn.dev/ozapon/articles/gmo-fx-hmac-sign-path) にあります。本記事はその先、**参照系の約定取得**で空配列化した話です

## 背景

GMOコインFX の API を使う自動売買システム(Python / AWS Lambda)では、定時の手仕舞い後に約定一覧を取得し、日次ノートを生成しています。

発注・取消・決済の経路では、[非常停止(Kill Switch)の設計](https://zenn.dev/ozapon/articles/gmo-fx-kill-switch-idempotency)どおり、ボディの `status` を検査していました。一方、**約定取得の GET** だけが HTTP ステータス中心の実装のまま残っていました。

公式ドキュメントでは、HTTP ステータスコード `200` とボディのステータスコード `0` が、それぞれ「処理が正常終了した場合」と説明されています(2026年8月時点で確認)。一方、実務では業務エラーでも HTTP 200 が返り、成否はボディの `status` で判定します。この挙動は認証エラー(`ERR-5010` / `ERR-5012`)の段階で既に経験していました。それでも約定 GET では、自分は HTTP 層の検査だけで足りると判断していました。

## 問題

ある朝の手仕舞い後、日次ノートが「約定件数 0 件 / ポジション未保有 / 未約定」といった文面になりました。

実際には、その取引日のエントリー約定と決済約定は完了していました。壊れていたのは発注・決済ロジックではなく、**約定取得とノート生成**だけです。取得結果が 0 件扱いになったため、下流が事実と反対の要約を書いてしまいました。

ログ上、約定取得は例外を出さず正常終了していました。HTTP は 200 で、呼び出し側は空のリストを受け取っていました。

## 原因

GMO Private API の業務エラーは、例えば次の形です。

```json
HTTP/1.1 200 OK

{
  "status": 1,
  "messages": [
    {
      "message_code": "ERR-5106",
      "message_string": "Invalid request parameter."
    }
  ],
  "responsetime": "2026-08-05T..."
}
```

`raise_for_status()` は HTTP 200 なので何もしません。成功時と違い、失敗時のボディには `data` がありません。

当時の素朴な実装は、おおよそ次のような流れでした。

```python
resp = client.private_get("/private/v1/latestExecutions", params=...)
resp.raise_for_status()  # HTTP 200 なら通過
body = resp.json()
# data が無い → 空配列として扱う
return body.get("data", {}).get("list", [])  # → []
```

結果として、**パラメータ不正などの業務エラーが「約定なし」に化けます**。「取得できなかった」と「約定が無かった」が区別できません。

監査すると、キャンセル・ファースト(取消・決済)側はすでに `raise_for_gmo_status` 相当の検査があり、**穴は約定系 GET だけでした**。認証の枝で知っていたはずの「ボディ status を見る」が、参照系の一部経路にだけ漏れていました。

なお、当時の問い合わせパラメータ自体にも誤りがありました(存在しないクエリを送っていた等)。それ自体も `ERR-5106` の誘因ですが、本記事の主眼は、そのエラーを**空配列として握りつぶしていた**点です。パラメータを直しても、ボディ status を見なければ同種の化けは再発します。

## 解決策

約定系 GET でも、HTTP 層とは独立にボディの `status` を検査します。

```python
def raise_for_gmo_status(resp, *, context: str) -> None:
    body = resp.json()
    status = body.get("status")
    if status is None or status == 0:
        return
    codes = tuple(
        m["message_code"]
        for m in body.get("messages", [])
        if isinstance(m, dict) and "message_code" in m
    )
    raise GmoApiError(
        f"GMO API error ({context}): status={status} codes={list(codes)}",
        status=status,
        message_codes=codes,
    )


def get_latest_executions(self, *, symbol: str, count: int = 100) -> list[dict]:
    resp = self.private_get("/private/v1/latestExecutions", params={
        "symbol": symbol,
        "count": count,
    })
    resp.raise_for_status()
    raise_for_gmo_status(resp, context="latestExecutions")  # 必須
    data = resp.json().get("data")
    # ... list を取り出す
```

HTTP 200 + `status: 1` は例外になり、空配列にはなりません。回帰テストでは「HTTP 200 の業務エラーで `GmoApiError` になり、空リストに化けない」ことを固定しています。

加えて、約定取得だけは**同一 invocation 内で短くリトライ**しています。この API で取れる範囲は直近に限られ、翌日の定時再実行では取り逃した明細を回収できないためです。読み取り専用の GET だから再送してよい、という前提付きです。発注系 POST には通しません。

主取得が最終的に失敗したらノートを生成せず失敗側に倒します。「約定 0 件のノート」を成功扱いで残さないためです。

## まとめ

HTTP 200 だけでは業務エラーを取りこぼします。`data` 欠落を空配列に倒すと、「取得失敗」が「約定なし」に化け、下流の要約が事実と反対になります。この2つは混ぜません。

発注側で検査済みでも、参照系の一部だけ穴が残ることがあります。経路横断で監査する、というのが持ち帰る判断です。

認証の HTTP 200 業務エラーは [ERR-5010](https://zenn.dev/ozapon/articles/gmo-fx-hmac-sign-path) / [ERR-5012](https://zenn.dev/ozapon/articles/gmo-fx-err5012-ipv6)、発注ボディの型は [ERR-5105](https://zenn.dev/ozapon/articles/gmo-fx-err5105-ifo-size)、決済シーケンス側の body status 検査は [Kill Switch の冪等設計](https://zenn.dev/ozapon/articles/gmo-fx-kill-switch-idempotency)、レート制御は [レートリミット設計](https://zenn.dev/ozapon/articles/gmo-fx-rate-limiter) に書いています。
