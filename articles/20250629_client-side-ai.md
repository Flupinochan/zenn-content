---
title: "Client Side AI×Google拡張機能"
emoji: "🕊️"
type: "tech"
topics: ["chrome拡張", "ai", "wxt", "react", "typescript"]
published: true
---

## はじめに

[Chrome 138](https://developer.chrome.com/docs/ai/built-in-apis?hl=ja) からブラウザに生成AI (Gemini Nano) が組み込み可能になりました!

以下のような翻訳Google拡張機能を作成しました!

![](/images/20250629_client-side-ai/output.gif)

## 環境

- React
- WXT

以下でGoogle拡張機能の開発環境をセットアップします

```bash
npx wxt@latest init
```

https://wxt.dev/guide/installation.html

## 機能

| 機能          | 用途                                                | 補足                                                         |
| ------------- | --------------------------------------------------- | ------------------------------------------------------------ |
| background.js | 拡張機能インストール時に生成AIのModelをダウンロード | Service Worker                                               |
| content.js    | 選択した文字列を翻訳                                | 訪れたWebブラウザに挿入されるJavaScript                      |
| popup.html    | 翻訳先の言語設定を保存                              | Google拡張機能のアイコンを選択した際に表示されるポップアップ |

## 使用するJavaScript API

| JavaScript API                                                                        | 用途                                       |
| ------------------------------------------------------------------------------------- | ------------------------------------------ |
| [Selection](https://developer.mozilla.org/ja/docs/Web/API/Selection)                  | カーソルで青色に選択した文字列や位置を取得 |
| [LanguageDetector](https://developer.mozilla.org/en-US/docs/Web/API/LanguageDetector) | 選択した文字列の言語を取得                 |
| [Translator](https://developer.mozilla.org/en-US/docs/Web/API/Translator)             | 選択した文字列を翻訳                       |
| [Storage](https://wxt.dev/storage.html)                                               | 翻訳先の言語設定をlocal storageに保存      |

## 実行例

### Selection

```javascript
document.addEventListener("mouseup", () => {
  const selection = window.getSelection();
  console.log(`選択した文字列: ${selection.toString()}`);
});
```

![](/images/20250629_client-side-ai/3.png)

### LanguageDetector

```javascript
const availability = await LanguageDetector.availability();
console.log(`利用可能状態: ${availability}`);
const detector = await LanguageDetector.create({
  monitor(m) {
    m.addEventListener('downloadprogress', (e) => {
      console.log(`Downloaded ${e.loaded * 100}%`);
    });
  },
});
const someUserText = 'Where is the next bus stop, please?';
const results = await detector.detect(someUserText);
console.log(`言語判定結果: ${results[0].detectedLanguage}`);
```

![](/images/20250629_client-side-ai/1.png)

### Translator

```javascript
const availability = await Translator.availability({
  sourceLanguage: 'en',
  targetLanguage: 'ja',
});
console.log(`利用可能状態: ${availability}`);
const translator = await Translator.create({
  sourceLanguage: 'en',
  targetLanguage: 'ja',
  monitor(m) {
    m.addEventListener('downloadprogress', (e) => {
      console.log(`Downloaded ${e.loaded * 100}%`);
    });
  },
});
const result = await translator.translate('Where is the next bus stop, please?');
console.log(`翻訳結果: ${result}`);
```

![](/images/20250629_client-side-ai/2.png)

## サンプルGoogle拡張機能

https://chromewebstore.google.com/detail/selectiontranslator/ckgmmdgflpnffbnkfoamlgfhmafidfmg?authuser=0&hl=ja

## サンプルコード

https://github.com/Flupinochan/GoogleExtension/tree/master/selection-translator/entrypoints

## おわりに

画像や音声が入力可能なマルチモーダル対応予定のPrompt APIも楽しみですね!