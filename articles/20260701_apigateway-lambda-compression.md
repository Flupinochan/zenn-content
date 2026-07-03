---
title: "API Gateway/LambdaでResponse BodyをGZIP圧縮して軽量化"
emoji: "🥀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["apigateway", "lambda", "python", "aws"]
published: true
---

## はじめに

突然、API GatewayとLambdaがエラーを返却するようになりました

Lambdaのエラーログは以下です

```json
{
    "errorMessage": "Exceeded maximum allowed payload size (6291556 bytes).",
    "errorType": "RequestEntityTooLarge"
}
```

このエラーは、Lambdaの最大レスポンスサイズ `6 MB` 制限によるエラーです
API GatewayやLambdaには、最大レスポンスサイズがクォータとして存在しています

対応策の一つとして、JSONレスポンスのフィールドを削って軽量化することを検討しました
しかし、様々なところで利用されているAPIだったため、既存のAPIは修正せず、既存のAPIを呼び出すラッパーとなる新規APIを定義し、新規APIでレスポンスのフィールドを削る、という対応をしました

ただし、これはあくまでも一時的な回避策で、他にも数百~数千のAPIが存在するため、今後も同様なエラーが様々なAPIで発生しそうでした
共通の対応策として、本ブログで **レスポンスをGZIP圧縮して返却する方法** を紹介いたします

実装が簡単な上に、レスポンスサイズが半減することが見込めます!!

:::message
本記事では `レスポンスサイズ` と `ペイロードサイズ` を同じ意味で使用しています
表記ゆれがある点はご了承ください
:::

## 前提

- API Gateway、Lambdaを知っている方
- AWS CDKについてなんとなく分かる方
- Pythonについてなんとなく分かる方

## 対象者

- API GatewayとLambdaのサーバレス構成を利用している方
- レスポンスをGZIP圧縮して返却することで、最大レスポンスサイズのクォータ超過を回避したい方

## 設定・コード例

@[card](https://github.com/Flupinochan/api-gateway-lambda-compression-sample)

### 1. API Gatewayで binaryMediaTypes を設定

以下は、AWS CDKのコード例です

```ts
const api = new apigateway.RestApi(this, "CompressionTestApi", {
    restApiName: "compression-test-api",
    // Lambda の response body は文字列のみ返却可能なため、バイナリは base64 エンコードして返却する必要がある
    // API Gateway はリクエストの Accept ヘッダーが binaryMediaTypes にマッチし、かつ isBase64Encoded: true のときに
    // response body を base64 デコードしてバイナリとしてクライアントへ返却する
    // */* により Accept ヘッダーの値を問わず常にマッチさせているが適切に設定しても良い
    // binaryMediaTypes はリクエスト/レスポンス両方に適用される
    binaryMediaTypes: ["*/*"],
    //...省略
```

コンソール画面からの設定箇所

![binary-media-types](/images/20260701_apigateway-lambda-compression/api-gateway-binary-media-types.png)

### 2. Lambdaでレスポンスを 圧縮・エンコード して返却

```python
# gzip.compress()はbytesしか受け付けないため、JSON文字列をUTF-8のbytesに変換
payload: bytes = json.dumps(
    {
        "message": "This is a gzip compressed binary response",
    },
    ensure_ascii=False,
).encode("utf-8")

# gzip圧縮
compressed: bytes = gzip.compress(payload, compresslevel=6)

# base64でエンコード
encoded: str = base64.b64encode(compressed).decode("ascii")

return {
    "statusCode": 200,
    "headers": {
        "Content-Type": "application/json",
        "Content-Encoding": "gzip",
    },
    "body": encoded,
    # Trueで返却
    "isBase64Encoded": True,
}
```

## おわりに

CloudFrontで自動的に圧縮されるからパフォーマンスは問題ない! と思っていましたが、そもそも圧縮前のデータに対して制限を受けることもあるんですね...

今年はバグ対応やエラー対応に関するブログ記事を投稿していく予定です! (って、もう7月ですねw)

もしかしたら年末にまとめて1つの記事にまとめるかもしれません

## 補足・参考

- [API Gateway のクォータ](https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/api-gateway-execution-service-limits-table.html)
  - ペイロードサイズは `10 MB` まで
- [Lambda のクォータ](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/gettingstarted-limits.html)
  - 通常 (Buffered) のリクエストとレスポンス は `6 MB (同期)` / `1 MB (非同期)` まで
  - ストリーミングレスポンス (Lambda関数URL/API Gatewayプロキシ統合、同期のみ) は `200 MB` まで
- [API Gateway のバイナリメディアタイプ](https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/api-gateway-payload-encodings.html#api-gateway-payload-encodings-proxy)
  - Lambdaプロキシ統合のAPI Gatewayではbase64でエンコードすることでGZIP (バイナリ) で圧縮してレスポンス可能
- [AWS公式ブログ](https://aws.amazon.com/jp/blogs/compute/optimizing-network-footprint-in-serverless-applications/)
  - [参考コード](https://github.com/aws-samples/lambda-with-compression/blob/main/lambda/get-gzip/index.mjs)
- [CloudFrontの圧縮条件](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/ServingCompressedFiles.html#compressed-content-cloudfront-notes)
  - オリジンからのレスポンスに `Content-Encoding` ヘッダーが含まれている場合は圧縮しない (圧縮が重複することはない)
- [FastAPI GZipMiddleware](https://fastapi.tiangolo.com/ja/advanced/middleware/#gzipmiddleware)
  - リスクは分かりませんが、ミドルウェアでまとめてGZIP化すると楽かもしれません

## 追記

現場では、ミドルウェア等による一括圧縮処理ではなく、各APIごとに個別に圧縮処理を設定することにしました

大きく分けて以下2点のリスクがあったためです

1. CPUチューニングコストの増加
   - 全APIに一括で圧縮処理を入れるとCPUを増やす必要があるのか調査するのに時間がかかりそうだったため
   - 今回設定したAPIでは、ほとんどCPUに影響はなかったため、特にCPUを増やしませんでした
2. 圧縮処理時間の増加によるAPI Gateway Timeoutエラー
   - 圧縮処理によって処理時間が伸び、API GatewayのTimeout 30秒 を超えてしまうAPIが出てくるかもしれないため
   - 現場では、20秒~くらいの処理が多く、圧縮処理を入れることで 30秒 を超える可能性があったためです
