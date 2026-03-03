---
title: "AmplifyでCSSやJSファイルを圧縮配信する方法"
emoji: "😺"
type: "tech"
topics: ["amplify", "cloudfront", "aws", "html", "typescript"]
published: true
---

## はじめに

前回に引き続き、`Lighthouse` でWebサイトのパフォーマンスを改善中です!

今回は、データ転送量を削減し、画面の初期表示を速くするために、CSSやJSファイルを圧縮して配信する方法を解説します

## 結論

AmplifyではデフォルトでCSSやJSファイルが圧縮されるため、特別な設定は不要!

※AmplifyはCloudFrontを利用して自動的に圧縮し配信しています

以降は、自分用のメモとして残しますが、参考になれば幸いです

## 圧縮して配信する際のフロー

1. ビルドしたHTML・JS・CSSファイルを圧縮し、サーバで圧縮ファイルを配信するよう設定
2. クライアント (ブラウザ) が `Accept-Encoding` ヘッダーで対応可能な圧縮方式を指定してサーバにリクエスト
3. サーバが対応する圧縮ファイルを選択し `Content-Encoding` ヘッダーに圧縮方式指定してレスポンス
4. ブラウザ側が `Content-Encoding` に従って解凍し、ページを表示

## 主な圧縮方式

一般的には以下2種類が使われています

圧縮率の高い `Brotli` が推奨されています

1. Brotli (.br)
2. Gzip (.gz)

![brotli](/images/20260303_amplify-compression/1.png)

@[card](https://web.dev/articles/optimizing-content-efficiency-optimize-encoding-and-transfer?hl=ja#text_compression_with_compression_algorithms)

配信時は以下の3種類を用意しておくと互換性が高くなります

古いブラウザでは新しいBrotliに対応していない場合があるためです

1. 圧縮してないオリジナルファイル
2. Brotliで圧縮したファイル
3. Gzipで圧縮したファイル

## ビルド時に圧縮する方法

:::message
Amplify (CloudFront) を利用する場合は、自動的に圧縮されるため設定は不要です
:::

手動で圧縮する場合、Viteの場合は `vite-plugin-compression` というpluginを利用すると簡単に圧縮できます

※また、`rollup-plugin-visualizer` を使うと圧縮後のサイズを可視化可能です

```typescript:vite.config.ts
import { visualizer } from "rollup-plugin-visualizer";
import { defineConfig } from "vite";
import viteCompression from "vite-plugin-compression";

export default defineConfig({
  plugins: [
    visualizer({
      template: "treemap",
      open: true,
      gzipSize: true,
      brotliSize: true,
      filename: "./tmp/stats.html",
    }),
    viteCompression({
      algorithm: "brotliCompress",
      ext: ".br",
      deleteOriginFile: false,
      threshold: 1024,
    }),
    viteCompression({
      algorithm: "gzip",
      ext: ".gz",
      deleteOriginFile: false,
      threshold: 1024,
    }),
  ],
});
```

試したところ、確かに `Brotli` の方がファイルサイズが小さかったです

![vite-compression](/images/20260303_amplify-compression/2.png)

## 圧縮したファイルを配信する方法

### Content-Encoding ヘッダーを指定

サーバ側で圧縮ファイルを配信する場合、`Content-Encoding` ヘッダーに圧縮方式を指定します

ブラウザはこのヘッダーに従って自動的に解凍します

```
Content-Encoding: gzip
```

@[card](https://developer.mozilla.org/ja/docs/Web/HTTP/Reference/Headers/Content-Encoding)

### CloudFrontの場合

1. CloudFront Policyで設定

以下はAmplifyに適用されているPolicyです

圧縮サポートを有効にします

![cloudfront-policy](/images/20260303_amplify-compression/3.png)

2. CloudFront Behaviorで設定

自動的に圧縮して配信されるように設定します

![cloudfront-behavior](/images/20260303_amplify-compression/4.png)

@[card](https://docs.aws.amazon.com/ja_jp/amplify/latest/userguide/cache-configuration-type.html)

@[card](https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/ServingCompressedFiles.html)

## 圧縮して配信されているか確認する方法

Chrome DevToolsでレスポンスヘッダーの `Content-Encoding` を確認します

※表示されていない場合は `右クリック` から表示項目を追加してください

![content-encoding](/images/20260303_amplify-compression/5.png)

## ブラウザが対応している圧縮方式を確認する方法

リクエストヘッダーの `Accept-Encoding` を確認します

サーバはこの情報をもとに圧縮方式を選択して配信します

普段あまり気にしていないですが、デフォルトで圧縮したファイルを選択するようになっているということです

![accept-encoding](/images/20260303_amplify-compression/6.png)

@[card](https://developer.mozilla.org/ja/docs/Web/HTTP/Reference/Headers/Accept-Encoding)

## jpeg, png, mp3, mp4の圧縮について

画像や動画、音声ファイルはすでに圧縮されているため、圧縮不要です

## おわりに

パフォーマンス改善の結果、Lighthouseのスコアがオール100になりました

オール100になると紙吹雪のアニメーションが表示されました

ぜひ、みなさんも試してみてください

![Lighthouse](/images/20260303_amplify-compression/lighthouse.png)
