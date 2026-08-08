---
title: "GMOコインFX APIのERR-5105でハマった話 — ifoOrderのsizeは整数文字列"
emoji: "🔢"
type: "tech"
topics: ["gmocoin", "fx", "python", "api"]
published: false
---

## TL;DR

- `POST /private/v1/ifoOrder` で `firstSize` / `secondSize` を JSON number(`100`)のまま送ると、`ERR-5105: Request parameter include mismatch type.` で弾かれます
- 正解はワイヤ上の**整数文字列**(`"100"`)。公式パラメータ表の Type も `string` です(2026年8月時点で確認)
- Python 側でロット検証のために `int` を持つのは妥当ですが、JSON シリアライズで number のまま出すと型不一致になります
- Pydantic の `field_serializer` で「ドメインは int / ワイヤは str」を分離しました
- メッセージは "mismatch type" だけで、どのフィールドかが特定しにくい。制御下の最小ロット疎通で実測して確定しました

## 背景

GMOコインFX の API を使う自動売買システム(Python)で、IFD-OCO 発注(`POST /private/v1/ifoOrder`)を実装していました。

発注系は本番口座に直接作用するため、[誤発注の封じ込め設計](https://zenn.dev/ozapon/articles/gmo-fx-order-containment)どおり、まず無効ボディで必須フィールド名を確認し、その後に two-key を一時解除した**制御下の最小ロット疎通**で成功系の仕様を潰す、という順序で進めています。

認証まわりの `ERR-5010`(署名パス / Accept)や `ERR-5012`(IPv6 egress)は既に通過済みでした。本記事はその先、ボディの型で止まった話です。

## 問題

最小ロット(100 通貨)の IFD-OCO を送ったところ、HTTP 200 のまま次の業務エラーが返りました。

```json
HTTP/1.1 200 OK

{
  "status": 1,
  "messages": [
    {
      "message_code": "ERR-5105",
      "message_string": "Request parameter include mismatch type."
    }
  ],
  "responsetime": "2026-08-03T..."
}
```

GMO の Private API は、成否を HTTP ステータスではなくボディの `status` で返します。`status: 1` なので業務エラーです。

困ったのはメッセージの粒度です。"mismatch type" としか書かれておらず、**どのフィールドの型が違うのか**が分かりません。エラーコード一覧にも `ERR-5105` への記載はありませんでした(2026年8月時点で確認)。

数量が小さすぎる場合は別コード `ERR-5126`(Size is invalid.) が返ります。今回は最小ロットどおり送っているので、値の範囲ではなく**型**の問題だと切り分けました。

## 原因

リクエストボディでは、数量を JSON number で送っていました。

```json
{
  "symbol": "USD_JPY",
  "firstSide": "BUY",
  "firstExecutionType": "LIMIT",
  "firstSize": 100,
  "firstPrice": "156.400",
  "secondSize": 100,
  "secondLimitPrice": "156.800",
  "secondStopPrice": "156.000"
}
```

公式ドキュメントの IFDOCO パラメータ表では、`firstSize` / `secondSize` の Type はいずれも `string` です(2026年8月時点で確認)。サンプルも `"10000"` のような引用符付きです。

自分はドメインモデルで数量を `int` にしていました。ロット下限 100・100 倍数のバリデーションを数値として書きたいからです。ここまでは妥当でした。

ただし Pydantic / `json` の既定シリアライズでは `int` は JSON number になります。**公式が求める string と、自分がワイヤに載せた number が食い違った**のが原因です。

`"100"` に変えて同じボディを送ると通過し、`100`(number)だと再び `ERR-5105` になりました(2026-08-03、制御下の最小ロット疎通で実測)。価格フィールドの検証に進む前に短絡するため、number のままでは発注が1件も通りません。

## 解決策

ドメインは `int` のまま、ワイヤに出すときだけ整数文字列へ変換します。

```python
from pydantic import BaseModel, Field, field_serializer, field_validator

class IfoOrderRequest(BaseModel):
    first_size: int = Field(alias="firstSize")
    second_size: int = Field(alias="secondSize")
    # ... 他フィールドは省略

    @field_validator("first_size", "second_size")
    @classmethod
    def size_must_be_valid_lot(cls, v: int) -> int:
        if v < 100:
            raise ValueError(f"size は最小 100 通貨が必要です: {v}")
        if v % 100 != 0:
            raise ValueError(f"size は 100 の倍数でなければなりません: {v}")
        return v

    @field_serializer("first_size", "second_size")
    def serialize_size_as_string(self, v: int) -> str:
        # ワイヤだけ str。JSON number だと ERR-5105
        return str(v)
```

`model_dump_json(by_alias=True)` 後のボディは次のようになります。

```json
{
  "firstSize": "100",
  "secondSize": "100"
}
```

テストでは「ワイヤが文字列であること」と「Python 側は int のままロット検証が効くこと」の両方を固定しています。型を最初から `str` にすると、`"150"` のような 100 倍数でない値の検証を自分で再実装することになり、責務がぶれやすいためです。

なお、同じ疎通で成功レスポンスの `data` が注文オブジェクトの配列であること(IFD-OCO ならエントリー1件 + 決済2件)も確定しました。レスポンス側の `size` も文字列です。配列の扱いや保存すべき ID の話は、封じ込め記事の「制御下の最小ロット疎通」の文脈に譲ります。

## まとめ

- `ifoOrder` の `firstSize` / `secondSize` は**整数文字列**で送る。JSON number だと `ERR-5105`
- 公式パラメータ表の Type は `string`。サンプルどおりに引用符付きで送れば踏まない
- ドメインを `int` にするなら、シリアライズ境界で str 化を明示する(`field_serializer` など)
- `ERR-5105` はメッセージからフィールドを特定しにくい。数量不正(`ERR-5126`)と切り分けたうえで、型を疑う
- 副作用のある仕様確定は、[封じ込め](https://zenn.dev/ozapon/articles/gmo-fx-order-containment)を維持した制御下の疎通で行う

認証の枝は [ERR-5010](https://zenn.dev/ozapon/articles/gmo-fx-hmac-sign-path) / [ERR-5012](https://zenn.dev/ozapon/articles/gmo-fx-err5012-ipv6)、設計の幹は [封じ込め](https://zenn.dev/ozapon/articles/gmo-fx-order-containment) / [レートリミット](https://zenn.dev/ozapon/articles/gmo-fx-rate-limiter) / [Kill Switch 冪等](https://zenn.dev/ozapon/articles/gmo-fx-kill-switch-idempotency) に書いています。
