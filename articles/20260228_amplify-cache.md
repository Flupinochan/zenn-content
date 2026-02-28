---
title: "Amplifyでのキャッシュ設定"
emoji: "🐔"
type: "tech"
topics: ["amplify", "s3", "lighthouse", "cache", "aws"]
published: true
---

## はじめに

`Lighthouse` を利用してWebサイトのパフォーマンス改善に取り組み中です!  
キャッシュ設定について備忘録としてまとめました

## 対象者

- CSS, JS, png, mp3などのファイルをキャッシュしたい方
- AWS Amplifyでのキャッシュ設定方法を知りたい方

## 結論

### Amplify Hosting の場合

SPAのフロントエンド本体 (index.html, CSS, JSファイル等) のキャッシュ設定です

`amplify.yml` ファイルの `customHeaders` で設定します

@[card](https://docs.aws.amazon.com/ja_jp/amplify/latest/userguide/custom-header-YAML-format.html)

```yaml:amplify.yml
version: 1
backend:
  phases:
    build:
      commands:
        - npm ci --cache .npm --prefer-offline
        - npx ampx pipeline-deploy --branch $AWS_BRANCH --app-id $AWS_APP_ID
frontend:
  phases:
    build:
      commands:
        - npm run build
  artifacts:
    baseDirectory: dist
    files:
      - "**/*"
  cache:
    paths:
      - .npm/**/*
      - node_modules/**/*
  customHeaders:
    # キャッシュさせるファイル
    - pattern: "**/*.@(js|css|png|jpg|jpeg|gif|webp|svg|woff2|ico)"
      headers:
        - key: Cache-Control
          value: public, max-age=31536000, immutable
    # index.htmlだけ別指定
    - pattern: "/index.html"
      headers:
        - key: Cache-Control
          value: public, max-age=0, must-revalidate
```

### Amplify Storage の場合

追加でデプロイ可能なS3のキャッシュ設定です

`uploadData` メソッドでS3にアップロードする際に `options` で指定します

音楽再生アプリを作成しているのですが、mp3等の音楽ファイルをアップロードする際に指定しています

:::message
設定は反映されますが、署名付きURLのためかキャッシュされませんでした
:::

```typescript:typescript
await uploadData({
    data: blob,
    path: "xxx",
    options: {
        contentType: "yyy",
        cacheControl: "public, max-age=31536000, immutable",
    },
}).result;
```

![1.png](/images/20260228_amplify-cache/1.png)

## キャッシュ判定の仕組み

- ファイル名 (URL) 単位で設定し、ブラウザが判定する
  - `immutable`: キャッシュして `指定期間` はサーバに `問い合わせない`
  - `no-cache (max-age=0, must-revalidate)`: キャッシュはするが、再ダウンロードが必要かどうか `ファイルの更新日` 等をサーバに `問い合わせる`
  - `no-store`: キャッシュしないで常にサーバからダウンロードする

## ファイル別のキャッシュ設定

### Assets (CSS, JS, png) ファイルの場合

`immutable` + `max-age` を指定

- viteやwebpack等でビルドするとハッシュ値が含まれるファイル名になる
- ファイルに変更があればビルドした際にハッシュ値が変わり別ファイルとして扱われる
- ファイル名単位でキャッシュ設定が適用されるため、新しいファイルはキャッシュが適用されないことがポイント
- 期間指定でキャッシュさせて問題ないファイル

### Entry Point (index.html) ファイルの場合

`no-cache` を指定

- ファイル名が固定で変わらないため、`immutable` (期間) でキャッシュすると古いファイルが参照し続けられる
- `no-cache` はキャッシュするが、サーバに問い合わせて `Last-Modified` (ファイルの更新日) や `ETag` (ファイルのハッシュ値) に変更があれば再ダウンロードし、変更がなければキャッシュを利用する仕組み
- サーバ側でEtagのルール等は設定可能みたい

### 機密情報の場合

`no-store` を指定

- キャッシュさせない (データを保持しない) べき

## 補足

- **`no-cache` と `max-age=0, must-revalidate` の違い**
  - どちらも同じ意味
  - 「キャッシュしない」という意味ではなく、「キャッシュするが、毎回サーバに問い合わせて更新がなければダウンロードせずキャッシュを利用する」という意味
  - `no-cache` という名称は誤解を招きやすいため、意図が明確な `max-age=0, must-revalidate` を使用するほうが良いとされているみたいです

- dev toolsでのキャッシュ確認方法
  - グレーアウトされた200: `immutable` でキャッシュされたファイル
    - (memory cache) でサイズは無し、ネットワーク通信は発生していない
  - 304: `no-cache` でキャッシュされたファイル
    - サーバからbodyは取得しないが、headerは取得しているため少量のサイズがある
      - ネットワーク通信は発生しているが速い
    - ファイルの更新状態を確認するために、headerは必ずダウンロードされる
    - bodyはキャッシュされるが、headerはキャッシュされていないとも言える

![2.png](/images/20260228_amplify-cache/2.png)

## おわりに

Lighthouseを使用してあらためて気づいたのですが、UIコンポーネントライブラリを使用すると、たくさんのユーティリティクラス (m-2, d-flex等) を含んだ巨大なCSSファイルが含まれています

使用していないCSSが多く含まれるのですが、これを最適化することはできず、Lighthouseの警告を消すことができませんでした...

そのため、UIコンポーネントライブラリを使用する時点で、ある程度パフォーマンスが犠牲になっているということを理解させられました...

オフライン対応させたい方は [GETとPOSTのオフライン対応](https://zenn.dev/metalmental/articles/20250614_service-worker) もあわせてご覧ください

## 参考資料

@[card](https://web.dev/articles/love-your-cache?hl=ja)
