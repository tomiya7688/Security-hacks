# 画像のURLを入力させるだけで、なぜ社内のサーバーまで危険になるの？

> **「ユーザーが指定した画像を取得するだけ。どうして自分たちのサーバーが狙われるの？」**

## 「あなたの代わりに取得する」機能を想像しよう

プロフィール画像をURLで登録できるサービスがあるとします。

```text
利用者が画像のURLを入力
           ↓
サービスのサーバーがURLにアクセス
           ↓
画像を取得して保存
```

便利ですが、この仕組みには見落としがあります。**URLを選ぶのは利用者なのに、アクセスするのはサービスのサーバー**です。

サーバーは、普通の利用者のブラウザーから見えない社内の機器や、管理用のネットワークに接続できる場合があります。利用者が行き先を自由に指定できると、**そのサーバーを代理人として、許可するつもりのなかった相手に通信させられる**かもしれません。

これを**SSRF（サーバーサイドリクエストフォージェリ）**といいます。日本語では「サーバー側での要求の偽造」といった意味です。

## どうして起きる？

「入力されたURLにアクセスすればよい」と設計して、**どこへアクセスしてよいかの境界を決めていない**ためです。

例えば「`https://` で始まるなら安全」と考えてしまうと、通信が暗号化されていることしか分かりません。**行き先がサービスにとって許可された相手かは分かりません。**

また、URLの文字列を見たときの行き先と、通信時に実際に使われるネットワーク上の行き先が、常に同じとは限りません。

## やってみよう：URLの「本当のホスト名」を調べる

次はPython 3の実験です。**URLの文字列を読み取るだけで、外部への通信は一切行いません。** `.test` は説明用のドメインです。

```python
from urllib.parse import urlsplit

ALLOWED_HOST = "images.example.test"

def looks_expected(url):
    try:
        parsed = urlsplit(url)
        return (
            parsed.scheme == "https"
            and parsed.hostname == ALLOWED_HOST
            and parsed.port in (None, 443)
            and parsed.username is None
            and parsed.password is None
        )
    except ValueError:
        return False

samples = [
    "https://images.example.test/cat.png",
    "https://images.example.test.other.test/cat.png",
    "http://images.example.test/cat.png",
]

for url in samples:
    print(looks_expected(url), url)
```

結果：

```text
True https://images.example.test/cat.png
False https://images.example.test.other.test/cat.png
False http://images.example.test/cat.png
```

2番目のURLには許可した名前が含まれていますが、**別のホスト名**です。「文字列のどこかに `images.example.test` があればOK」という判定では見誤ります。

## どう防ぐ？

最初に考えるべきなのは、**利用者がURLを自由に指定する必要が本当にあるか**です。

- できるなら、画像を取得する行き先を**自分たちが管理するサービスだけ**に限定します。
- URLを使う必要があれば、プロトコル・ホスト名・ポートを正しく解析して、**許可した組み合わせだけ**を使います。
- **転送先が別の場所に変わる応答（リダイレクト）**を勝手に追いかけないようにします。
- 名前から実際のネットワークの接続先を調べる仕組み（**DNS**）も考慮し、**社内向け・管理用のネットワークへの通信をサーバー側で制限**します。
- 取得するデータの量や時間にも上限を設け、ファイルの種類も確かめます。

## ここまでは分からない

**上の `looks_expected()` は、あくまでURL文字列の違いを示す模型です。** この関数が `True` を返しても、「そのURLをサーバーから取得して安全」という意味にはなりません。

実際には、名前を調べた結果が後から変わる場合や、接続先が別のURLに転送される場合があります。**URLの見た目の検査だけでSSRFを防ごうとしないでください。** DNSと実際の接続先の制限、送信できるネットワークの制御などが必要です。

他人のサーバーや社内ネットワークで試す必要はありません。仕組みはこの**通信しない模型**だけでも理解できます。

## もっと知りたい人へ

- [OWASP：Server Side Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html)
- [MITRE：CWE-918（サーバー側での要求の偽造）](https://cwe.mitre.org/data/definitions/918.html)
- [Python公式：URLの解析](https://docs.python.org/3/library/urllib.parse.html)

[← 「なぜ起きる？」一覧に戻る](README.md)
