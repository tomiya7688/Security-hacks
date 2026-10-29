# ログインしているのに、なぜ他人の注文まで見えてしまうの？

> **「注文ページの番号を変えたら、別の人の注文が表示された。ログインしているから安全なはずでは？」**

## 「あなたは誰？」と「これはあなたのもの？」は別

あるサイトで、自分の注文のURLが次のようになっていたとします。

```text
https://shop.example.test/orders/7
```

最後の `7` は、どの注文を見るかを指定する番号です。

ここでサイトが「ログインしているか」だけを確認して、その番号の注文をそのまま返したらどうなるでしょう。**ログイン済みの別の人にも、注文7が見えてしまう**かもしれません。

誰がログインしているかを確かめることを**認証**、その人がその注文を見てもよいか確かめることを**認可**といいます。この問題では、**注文ごとの認可が抜けています**。

番号などを指定するだけで、見る権利のないデータに届いてしまう問題を**IDOR（不適切な直接オブジェクト参照）**と呼びます。「番号を変えられること」が問題なのではなく、**番号に対応するデータを使ってよいか調べないこと**が問題です。

## どうして起きる？

プログラムが、次のように作られているからです。

```text
悪い流れ
ログイン済み？ → はい → 指定された注文番号を探す → そのまま返す

必要な流れ
ログイン済み？ → はい → 指定された注文番号を探す
                              ↓
                         この人が見てよい注文？
                           ↓ はい
                         返してよい
```

## やってみよう：2人分の架空データだけで確認

以下は**ネットワーク通信を一切しない**、Python 3の実験です。

```python
orders = {
    7: {"owner": "aki", "item": "本"},
    8: {"owner": "mio", "item": "ノート"},
}

def unsafe_order(current_user, order_id):
    return orders.get(order_id)  # 誰の注文か見ていない

def safe_order(current_user, order_id):
    order = orders.get(order_id)
    if order is None or order["owner"] != current_user:
        return None
    return order

print("確認がない場合:", unsafe_order("aki", 8))
print("確認がある場合:", safe_order("aki", 8))
```

結果：

```text
確認がない場合: {'owner': 'mio', 'item': 'ノート'}
確認がある場合: None
```

最初の関数は、**akiさんの画面なのに、mioさんの注文を返しています**。後半は、注文の持ち主が一致しないため返しません。

## どう防ぐ？

- **データを読む・変更する・削除するたびに**、その利用者が操作してよい対象かサーバー側で確認します。
- 利用者の識別情報は、サーバーが確かめたログイン状態から取得します。画面から送られた `owner=aki` のような自己申告だけを信用しません。
- DB検索の時点から、「**この利用者に許される注文だけ**」に絞り込む設計も有効です。
- 自分が閲覧できる注文と、他人の注文で、結果が適切に異なることを**許可された検証環境**でテストします。

番号を長くランダムにしたり、画面のボタンを隠したりするだけでは対策になりません。番号が推測しにくくても、**他の経路で知られた番号に対して確認が抜けていれば危険**です。

## ここまでは分からない

現実には「家族で共有する注文」「会社の管理者だけ見てよい情報」などもあります。この実験の「持ち主が同じならOK」は**一番単純なモデル**です。実際のサービスでは、役割や共有設定を含むルールを設計する必要があります。

なお、SQLの組み立て方が安全でも、**認可の確認がなければIDORは残ります**。[入力が命令に化ける問題](sql-injection.md)とは別に対処してください。

## もっと知りたい人へ

- [OWASP：Insecure Direct Object Reference Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html)
- [OWASP：Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- [MITRE：CWE-639（外部入力による認可のすり抜け）](https://cwe.mitre.org/data/definitions/639.html)

[← 「なぜ起きる？」一覧に戻る](README.md)
