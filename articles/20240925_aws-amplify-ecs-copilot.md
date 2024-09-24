---
title: "AWS ECS Copilot CLIのStatic Siteは、SSRに対応していないので気を付けよう!"
emoji: "🌟"
type: "tech"
topics: ["AWS", "Amplify", "ECS", "Copilot", "nextjs"]
published: true
---

## 背景
以前は、静的なReactアプリを自力で作成した「CloudFront + S3」にデプロイしていました

しかし、最近は、Next.jsでSSRを使用していて、「そのままデプロイしたら動かないじゃん!」と気づきました…

なぜなら、「CloudFront + S3」ではSSRに対応していないからです


## 解決策
#### 1. AWS Amplifyを使用する

「Amplifyの実態もCloudFront + S3だから動かないでしょ？」と思った方、違います！

動くんです! (　-`ω-)

https://docs.aws.amazon.com/ja_jp/amplify/latest/userguide/ssr-amplify-support.html
https://docs.aws.amazon.com/ja_jp/amplify/latest/userguide/ssr-supported-features.html#ssr-nextjs11-support

理由としては、「CloudFront + S3 + Lambda@Edge」のような構成になっているためです

具体的な仕組みは分かりませんが、S3のみでデプロイしているわけではないようです


#### 2. Next.jsでStatic Exports (SSG) を使用する

```js:next.config.mjs
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'export',
}

export default nextConfig;
```

これはSSRをやめる方法ですので、ご注意ください…

https://nextjs.org/docs/app/building-your-application/deploying/static-exports
https://nextjs.org/docs/app/building-your-application/deploying/static-exports#unsupported-features


#### 3. コンテナに移行しよう!

最近、NestJSを勉強したため、バックエンドのついでにフロントエンドもコンテナに移行しようと思いました

そこで、Copilot CLIでデプロイをしようとしてマニュアルを確認したところ、「Static Site」があり、これでいけるのでは？ と思い、試してみましたが、ダメでした…

https://aws.github.io/copilot-cli/ja/docs/manifest/static-site/

Copilot CLI (CloudFormation) で作成されたリソースを確認すると、Lambda@Edgeはありましたが、ただリダイレクトしているだけでした…

残念…

![](/images/20240925_aws-amplify-ecs-copilot/ecs-copilot-created-resource.png)

ただし、Copilot CLI自体は非常に便利で、簡単にコンテナをデプロイできるためオススメです!

個人利用ではサーバーレスの方がコストが低いため、使えればと思った次第です

※補足
Lambda@Edgeのコードは以下です
ChatGPTにコメントアウトしてもらいました

```js
function handler(event) {
    var request = event.request; // イベントからリクエストオブジェクトを取得
    var uri = request.uri; // リクエストのURIを取得

    // URIがスラッシュで終わる場合
    if (uri.endsWith('/')) {
        request.uri += 'index.html'; // URIの末尾に'index.html'を追加
    }
    // URIにドットが含まれていない場合（ファイル拡張子がない）
    else if (!uri.includes('.')) {
        request.uri += '/index.html'; // URIの末尾に'/index.html'を追加
    }
    
    return request; // 修正したリクエストオブジェクトを返す
}
```