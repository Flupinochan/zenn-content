---
title: "【View Transitions × Mask】GIFアニメーションでダークモード切替!!"
emoji: "🕸️"
type: "tech"
topics: ["html", "css", "javascript", "animation", "darkmode"]
published: true
---

## はじめに

@[card](https://elysiajs.com/)

ElysiaJSの公式サイトで実装されているダークモードの切り替え演出が面白かった!

ので共有したかった次第です

## 概要

GIFアニメーションでダークモードに切り替えるコード例を紹介します

- GIF画像を用いたmask
- View Transitionによる遷移

![dark-mode-gif](/images/20260418_gif-animation-dark-mode/dark-mode.gif)

## コード例

@[card](https://github.com/Flupinochan/dark-mode-transition/tree/master)

:::details コード抜粋
```html
<!doctype html>
<html lang="ja">
  <body>
    <h1>Hello World!</h1>
    <h2>maskとview-transitionで実現するダークモード切替アニメーション</h2>
    <button>ダークモード切替</button>
  </body>
</html>

<style>
  body {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    min-height: 100dvh;
  }

  .dark {
    background-color: #333;
    color: #fff;
  }

  button {
    padding: 0.5em 1em;
    font-size: 1.2em;
    cursor: pointer;
    color: inherit;
    background-color: transparent;
    border-radius: 4px;
  }

  ::view-transition-group(root) {
    z-index: 100;
  }
  ::view-transition-new(root),
  ::view-transition-old(root) {
    animation: scale 1.5s both;
  }
  ::view-transition-new(root) {
    mask: url("./dark-mode.gif") center / 0 no-repeat;
  }
  @keyframes scale {
    0% {
      mask-size: 0;
    }
    10% {
      mask-size: 25vmax;
    }
    90% {
      mask-size: 25vmax;
    }
    100% {
      mask-size: 750vmax;
    }
  }
</style>

<script>
  document.querySelector("button").addEventListener("click", async () => {
    const root = document.documentElement;
    await document.startViewTransition(() => {
      root.classList.toggle("dark");
    });
  });
</script>
```
:::

## おわりに

私も会社でElysiaJSようなユーモアのあるドキュメントサイトにしたいのですが、難しいですね...