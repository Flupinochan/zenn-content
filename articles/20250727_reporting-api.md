---
title: "Reporting APIでWebを監視してみた"
emoji: "🦭"
type: "tech"
topics: ["html", "amplify", "react", "typescript"]
published: true
---

## はじめに

最近は、Client Side AIの情報収集のために、[Chrome Platform Status](https://chromestatus.com/roadmap) でリリースされたJavaScript APIをチェックするようにしています

そこで [Reporting API](https://developer.chrome.com/docs/capabilities/web-apis/reporting-api?hl=ja) という `セキュリティ違反` や `非推奨API` の呼び出しをモニタリングする機能があったため試してみました!

## 仕組み

JavaScript APIですが、レポートの送信にはJavaScriptを利用するのではなく、HTTP Headerに送信先や送信対象を設定することで、エラー発生時に指定したEndpointに自動的にレポートがPOST送信されます

| Key                                      | Value                                   |
| ---------------------------------------- | --------------------------------------- |
| Reporting-Endpoints                      | レポートの送信先サーバの定義            |
| Content-Security-Policy-Report-Only      | 外部スクリプト・CSS・画像の不正読み込み |
| Cross-Origin-Embedder-Policy-Report-Only | CORP未設定のiframe・画像の埋め込み      |
| Cross-Origin-Opener-Policy-Report-Only   | 不正なページ間通信                      |
| Permissions-Policy-Report-Only           | カメラ・マイク・位置情報の不正利用      |

私の [ポートフォリオサイト](https://www.metalmental.net/) は、AWS Amplifyでデプロイしているため、以下のように設定しました

```yaml
customHeaders:
  - pattern: "**"
    headers:
      - key: Reporting-Endpoints
        value: default="https://metalmental.uriports.com/reports"
      - key: Content-Security-Policy-Report-Only
        value: default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self';
          object-src 'none'; report-to default
      - key: Cross-Origin-Embedder-Policy-Report-Only
        value: require-corp; report-to=default
      - key: Cross-Origin-Opener-Policy-Report-Only
        value: same-origin; report-to=default
      - key: Permissions-Policy-Report-Only
        value: camera=(), microphone=(), geolocation=(), payment=(); report-to=default
```

:::message
`Reporting-Endpoints` で送信先(default)を定義し、その他のKeyでdefaultに送信しています
また、非推奨APIの利用については、`Reporting-Endpoints` の設定のみで送信されるようです
:::

![](/images/20250727_reporting-api/4.png)

## 送信例

Chrome DevTools の `Application` タブから確認可能です

![](/images/20250727_reporting-api/1.png)

![](/images/20250727_reporting-api/2.png)

[URIports](https://www.uriports.com/) という監視ツールに送信し可視化してみました

外部のAPI、CSS、Google Fontの読み込みでCSPが発生していました

![](/images/20250727_reporting-api/3.png)

## 感想

ログが多く、コスト観点や可視化の必要性からデメリットが大きい気がします
そのためか、ネット上の情報も少ないです
開発環境で試してみるくらいがちょうどよいかもしれません
