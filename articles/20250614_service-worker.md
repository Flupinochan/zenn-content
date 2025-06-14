---
title: "GETとPOSTのオフライン対応"
emoji: "🐲"
type: "tech"
topics: ["contest2025ts", "serviceworker", "workbox", "react", "vite"]
published: true
---


## はじめに

`Service Worker`?、`Workbox`?、`キャッシュ戦略`? よく分からないけれど、脳死でGETとPOST全てをオフライン対応したい! そんな人向けの記事です

## 環境

- React
- Vite
- Workbox

以下でViteのプラグインおよびWorkboxがインストールできます

```bash
npm install -D vite-plugin-pwa @types/workbox-sw
```

## 設定例

https://github.com/Flupinochan/MyBlogV3/blob/main/src/service-worker/serviceWorker.ts

### vite.config.ts

```typescript
import { defineConfig } from 'vite'
import { VitePWA } from 'vite-plugin-pwa'

export default defineConfig({
  plugins: [
    VitePWA({
      // 開発環境でService Workerを有効化
      devOptions: {
        enabled: true,
        type: 'module',
      },
      // 別ファイルで定義したmanifest.jsonを使用
      strategies: 'injectManifest',
      // ServiceWorkerファイルがあるディレクトリパス
      srcDir: 'src/service-worker',
      // ServiceWorkerファイル名
      filename: 'serviceWorker.ts',
      injectManifest: {
        // キャッシュするファイル
        globPatterns: ['**/*.{js,css,html,ico,png,jpg,jpeg,svg,woff2,json,yaml,txt,webp}'],
        // キャッシュしないファイル
        globIgnores: ['**/*.gif'],
        // 最大キャッシュサイズ
        maximumFileSizeToCacheInBytes: 10 * 1024 * 1024,
      },
      // ServiceWorker自動登録設定
      registerType: 'autoUpdate',
    })
  ],
})
```

### serviceWorker.ts

```typescript
import { registerRoute } from 'workbox-routing';
import { NetworkFirst, NetworkOnly } from 'workbox-strategies';
import { CacheableResponsePlugin } from 'workbox-cacheable-response';
import { BackgroundSyncPlugin } from 'workbox-background-sync';
import { clientsClaim, setCacheNameDetails } from 'workbox-core';
import { cleanupOutdatedCaches, precacheAndRoute } from 'workbox-precaching';
import type { RouteMatchCallback, WorkboxPlugin } from 'workbox-core/types';

// 型定義
declare let self: ServiceWorkerGlobalScope

// precache名 ※vite.config.ts globPatternsで定義したファイルのキャッシュ名
setCacheNameDetails({
  prefix: 'metalmental',
  suffix: 'v0.0.2',
});
// 動的キャッシュ名
const cacheName = `metalmental-get-cache-002`;

// 即座にServiceWorkerを有効化
self.skipWaiting();
clientsClaim();
// 古いキャッシュを削除
cleanupOutdatedCaches();
// vite.config.ts globPatternsで定義したファイルをキャッシュ
precacheAndRoute(self.__WB_MANIFEST);


/////////
// GET //
/////////
// キャッシュ対象はGETメソッド
const matchCallback: RouteMatchCallback = ({ request }) => request.method === 'GET';
// 3秒以内に応答しなければキャッシュ応答
const networkTimeoutSeconds = 3;
// キャッシュ対象のHttpStatusCode
const statuses = [0, 200]

registerRoute(
  matchCallback,
  // Network優先、Offlineであればキャッシュで応答
  new NetworkFirst({
    networkTimeoutSeconds,
    cacheName,
    plugins: [
      new CacheableResponsePlugin({
        statuses
      }),
    ],
  })
);

//////////
// POST //
//////////
// Chrome推奨 (Braveだとエラーになりました)
const bgSyncPlugin = new BackgroundSyncPlugin('MetalMentalQueue', {
  maxRetentionTime: 24 * 60, // 24時間保存

});
const statusPlugin: WorkboxPlugin = {
  // ネットワークエラーなどで例外発生時に実行
  handlerDidError: async () => {
    // HttpStatusCodeを202でレスポンスを返却
    return new Response(
      JSON.stringify({ error: 'POSTリクエストでエラーが発生しました。後で再試行されます。' }),
      {
        status: 202,
        headers: { 'Content-Type': 'application/json' }
      }
    );
  }
};

registerRoute(
  /.*/, // 正規表現で全パス指定
  new NetworkOnly({
    plugins: [bgSyncPlugin, statusPlugin],
  }),
  'POST'
);
```

manifest.json は省略します
https://developer.mozilla.org/ja/docs/Mozilla/Add-ons/WebExtensions/manifest.json

## 解説

以下3サイトをもとに解説します

https://developer.chrome.com/docs/workbox/service-worker-overview?hl=ja
https://developer.chrome.com/docs/workbox/modules/workbox-background-sync?hl=ja
https://vite-pwa-org.netlify.app/guide/

### Service Workerとは

- Webブラウザ標準機能
- WebブラウザとWebサーバ間のプロキシとして動作するJavaScript
- Google拡張機能でも使用される
- キャッシュストレージやIndexedDBを使用したオフライン対応が可能
- fetchやsyncなどのイベントを監視可能

F12キー (Google Developer Tools) で登録されているService Workerが確認できます

![](/images/20250614_service-worker/1.png)

### Workbox とは

- Service Workerの設定を簡単にするJavaScriptライブラリ

Service WorkerはWebブラウザ標準機能のため、ライブラリを一切使用せずに使用することも可能ですが、Workboxを使用することが一般的なようです

https://developer.mozilla.org/ja/docs/Web/API/Service_Worker_API/Using_Service_Workers

### キャッシュ戦略

2パターンを以下に示します

![](/images/20250614_service-worker/2.png)

- **Cache first, falling back to network**
  1. キャッシュがある場合は、キャッシュを使用
  2. キャッシュがない場合は、Webサーバにアクセス
  3. Webサーバからのレスポンスをキャッシュしつつ、返却

Webサーバの負荷を減らし、画面表示を高速化する戦略です
もちろんオフライン対応です

![](/images/20250614_service-worker/3.png)

- **Network first, falling back to cache**
  1. 通常はWebサーバへアクセスし、レスポンスをキャッシュしつつ、返却
  2. 後でオフラインになった場合は、キャッシュで応答

オフラインになった場合の保険としてキャッシュしておく戦略です
キャッシュの影響で古い情報が表示されないことはメリットです

### ローカルでの開発、デバッグ手順

![](/images/20250614_service-worker/4.png)

以下の機能が便利です

- `オフライン`: チェックするとオフラインに
- `再読み込み時に更新`: WebブラウザをリロードするとService Workerが再読み込みされる
- `ネットワークのバイパス`: キャッシュを使用せずにWebサーバからのレスポンスを返却
- `Ctrl + Shift + R`: キャッシュを使用せずにWebブラウザをリロード
- `同期`: IndexedDBに保存されたPOST処理を再処理

![](/images/20250614_service-worker/5.png)

`ネットワーク` タブの `サイズ` カラムからServiceWorkerがキャッシュで応答したかどうかが確認可能です

私の設定はNetwork firstのため、オフラインの場合は以下のように動作していることが読み取れます

1. Webサーバへのリクエストが失敗
2. 代わりにServiceWorkerがキャッシュをレスポンス

![](/images/20250614_service-worker/6.png)

- `キャッシュ ストレージ`: GETリクエストのキャッシュサイズ
- `Service Worker`: Service Worker JavaScriptのサイズ
- `IndexedDB`: POSTリクエストのキューサイズ

開発環境のため多めですが、それでも多すぎる状態です...w
2MB以下にすべきです

![](/images/20250614_service-worker/7.png)
![](/images/20250614_service-worker/8.png)

オフライン時にPOSTリクエストがキュー(IndexedDB)に保存され

![](/images/20250614_service-worker/9.png)

オンライン時に再送された際のコンソールログです
![](/images/20250614_service-worker/10.png)

### レシピ

https://developer.chrome.com/docs/workbox/modules/workbox-recipes?hl=ja

最も簡単にキャッシュを実装する方法です。分からなかった場合は、これを参考にしてみるといいかもです

## 補足

- Braveで上手く動作しないことがありました。Chromeが安定します。
- Google Developer Toolsでofflineを設定すると上手く動作しないことがありました。Windows OS側の設定でWifiやEthernetを無効化する方が確実でした。
- 自分のようにGETやPOSTメソッドを対象に全てキャッシュ、キューに入れることは推奨されないため気を付けてください。一般的には、特定のファイルやリクエストパスのみをキャッシュします。

![](/images/20250614_service-worker/11.png)

## サンプル

ネットを切断して試してみてください!

https://www.metalmental.net/