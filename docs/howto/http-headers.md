# WebサイトのHTTPヘッダーを見てみる

> **所要時間：2分 / 必要なもの：curl**

「HTTPSになっているから安心」で終わりにせず、サーバーがどんなヘッダーを返すか見てみましょう。**読み取りだけの確認**ですが、アクセス先は自分が管理しているサイトや、閲覧が許可されたサイトにしましょう。

## まず試す

Linux / macOS のターミナルで：

```sh
curl -sS -D - -o /dev/null https://example.com/
```

Windows の PowerShell で：

```powershell
curl.exe -sS -D - -o NUL https://example.com/
```

- `-D -`：レスポンスヘッダーを画面に表示
- `-o /dev/null`（Windowsでは `NUL`）：レスポンス本文は保存しない
- `-sS`：進捗表示を抑え、エラーは表示

※ `curl -I` は HEAD リクエストになるため、GET と応答が異なるサイトもあります。上の例は GET で確認します。

## どこを見る？

実際の表示内容はサイトによって異なります。次の項目を探してみましょう。

| 項目 | ざっくり意味 | 見るポイント |
| --- | --- | --- |
| `HTTP/2 200` など | サーバーからの応答結果 | `200` は成功、`301` / `302` はリダイレクトなど |
| `Strict-Transport-Security` | HTTPS の利用をブラウザーに促す設定（HSTS） | HTTPSの応答に設定されているか |
| `Content-Security-Policy` | 読み込めるスクリプト等を制限する仕組み（CSP） | 値がそのサイトの構成に合っているか |
| `X-Content-Type-Options: nosniff` | ファイルの種類を勝手に推測する動作を抑制 | `nosniff` があるか |
| `Set-Cookie` | Cookie の設定 | セッション用Cookieなら `Secure`、`HttpOnly`、`SameSite` など |

**注意：ヘッダーがあるだけで安全とは言い切れません。** 必要な設定はサイトの用途で変わりますし、設定値が不適切な場合もあります。

## リダイレクト先まで確認したい

```sh
curl -sS -L -D - -o /dev/null https://example.com/
```

`-L` を付けるとリダイレクトを辿ります。この場合、**途中の応答ヘッダーも複数表示される**ことがあります。Windowsでは `/dev/null` を `NUL` に置き換えてください。

## よくあるつまずき

- **`301` / `302` しか見えない**：まず `Location` を確認。必要なら `-L`。
- **`curl` が見つからない**：インストール状況を確認。PowerShell では `curl.exe` を使うと別コマンドとの混同を防げます。
- **HSTS や CSP がない**：それだけで脆弱性と断定しない。対象の設計や運用要件と照らして判断。
- **Cookie が表示されない**：その応答で Cookie を発行していないだけかもしれません。ログイン時など応答条件で変わります。

## さらに読む

- [セキュリティの基本](../basics.md)
- [Q&A](../qa.md)
- [curl の公式マニュアル](https://curl.se/docs/manpage.html)
- [MDN: HTTP](https://developer.mozilla.org/ja/docs/Web/HTTP)

> **この手法でわかること**：レスポンスに何が設定されているか。**これだけではわからないこと**：サイト全体の安全性や、実際の攻撃耐性。
