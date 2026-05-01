---
title: "View Transitionでテーブルをアニメーション!!"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["html", "css", "javascript", "typescript", "react"]
published: true
---

## はじめに

[前回](https://zenn.dev/metalmental/articles/20260418_gif-animation-dark-mode) はView TransitionとGIFアニメーションを組み合わせました

今回は、テーブルと組み合わせて、ソート処理にView Transitionによるアニメーションを適用してみます！

![debug1](/images/20260502_view-transition-table/animation.gif)

## 前提

- React (useState, useRef) を知っていること
- CSS Animation (fade-in, fade-out) を知っていること
- なんとなくView Transitionを知っていること
- Chrome 147以降の最新のブラウザであること

## 対象者

- View Transitionでテーブルをアニメーションしたい方

## コード例

@[card](https://github.com/Flupinochan/view-transition-table/blob/master/src/App.tsx)

上記に完全なコードがありますが、以下に抜粋しました

:::details CSS
```css:App.css
/* 移動 */
::view-transition-group(.row-animation) {
  animation-duration: 1s;
  animation-timing-function: cubic-bezier(0.76, 0, 0.24, 1);
}

/* old/newで個別に制御するため無効化 */
::view-transition-image-pair(.row-animation) {
  animation: none;
}

/* フェードアウト */
::view-transition-old(.row-animation) {
  animation: fade-out 1s ease forwards;
}

/* フェードイン */
::view-transition-new(.row-animation) {
  animation: fade-in 1s ease forwards;
}

@keyframes fade-out {
  from {
    opacity: 1;
  }
  to {
    opacity: 0;
  }
}

@keyframes fade-in {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}
```
:::

:::details React
```ts:App.tsx
import { useRef, useState } from "react";
import "./App.css";
import { flushSync } from "react-dom";

function App() {
  const initialData = [
    {
      id: 1,
      name: "Alice",
      age: 25,
      email: "alice@example.com",
      country: "Japan",
    },
    { id: 2, name: "Bob", age: 30, email: "bob@example.com", country: "USA" },
    {
      id: 3,
      name: "Charlie",
      age: 28,
      email: "charlie@example.com",
      country: "UK",
    },
    {
      id: 4,
      name: "Dave",
      age: 35,
      email: "dave@example.com",
      country: "Canada",
    },
    {
      id: 5,
      name: "Eve",
      age: 22,
      email: "eve@example.com",
      country: "Germany",
    },
  ];

  const [data, setData] = useState(initialData);
  const tbodyRef = useRef<HTMLTableSectionElement>(null);
  const [isTransitioning, setIsTransitioning] = useState(false);

  // テーブルレコードのシャッフル処理 (生成AIで生成したコード)
  const shuffle = () => {
    const updated = [...data];

    // --- ランダム削除（50%の確率）
    if (updated.length > 0 && Math.random() < 0.5) {
      const removeIndex = Math.floor(Math.random() * updated.length);
      updated.splice(removeIndex, 1);
    }

    // --- ランダム追加（50%の確率）
    if (Math.random() < 0.5) {
      const newItem = {
        id: Date.now(), // 一意ID
        name: "NewUser",
        age: Math.floor(Math.random() * 40) + 20,
        email: "new@example.com",
        country: "Unknown",
      };
      updated.push(newItem);
    }

    // --- シャッフル（Fisher–Yates）
    for (let i = updated.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [updated[i], updated[j]] = [updated[j], updated[i]];
    }

    setData(updated);
  };

  const onClick = async () => {
    // 最新のブラウザではdocumentレベルだけでなく、elementレベルでもView Transition APIが利用可能ですが
    // 型定義はまだdocumentにしか対応していないため、anyでキャストして直接呼び出しています
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    const tbody = tbodyRef.current as any;
    if (!tbody || !tbody.startViewTransition) {
      throw new Error("View Transition API is not supported in this browser.");
    }

    try {
      // アニメーション開始前にview-transition-nameを付与
      setIsTransitioning(true);
      // View Transitionアニメーション開始
      await tbody.startViewTransition(() => {
        // DOMを強制的に再レンダリング (Reactを利用している場合のみ)
        flushSync(() => {
          // データ更新
          shuffle();
        });
      }).finished;
    } finally {
      // アニメーション完了後にview-transition-nameを削除
      setIsTransitioning(false);
    }
  };

  return (
    <>
      <button onClick={onClick} disabled={isTransitioning}>
        ランダム並び替え
      </button>

      {/* <table>タグはレイアウト再設計するため、View Transition APIを利用すると挙動が不安定
          そのため、divで代替。セマンティクスを保つのであればrole属性を利用すること */}
      {/* table */}
      <div>
        {/* header */}
        <div style={{ display: "flex" }}>
          <div style={{ flex: 1 }}>ID</div>
          <div style={{ flex: 1 }}>名前</div>
          <div style={{ flex: 1 }}>年齢</div>
          <div style={{ flex: 1 }}>メール</div>
          <div style={{ flex: 1 }}>国</div>
        </div>

        {/* body */}
        <div ref={tbodyRef}>
          {data.map((row) => (
            // row
            <div
              key={row.id}
              style={{
                display: "flex",
                // 色々な個所でView Transition APIを利用する場合は、競合が起きます
                // アニメーションする時だけ、view-transition-nameを付与し、
                // アニメーション完了後に削除しておくと安全です
                viewTransitionClass: isTransitioning ? "row-animation" : "none",
                // shuffle前のDOMとshuffle後のDOMを関連付けてアニメーションさせるための識別子
                // ユニークである必要がある
                // oldとnewでスナップショットが取られて、DOM位置やサイズなどをアニメーション可能
                viewTransitionName: isTransitioning ? `row-${row.id}` : "none",
              }}
            >
              {/* cell */}
              <div style={{ flex: 1 }}>{row.id}</div>
              <div style={{ flex: 1 }}>{row.name}</div>
              <div style={{ flex: 1 }}>{row.age}</div>
              <div style={{ flex: 1 }}>{row.email}</div>
              <div style={{ flex: 1 }}>{row.country}</div>
            </div>
          ))}
        </div>
      </div>
    </>
  );
}

export default App;
```
:::

## View Transition APIのデバッグ方法について

### 1. Chrome DevToolsでAnimationsを開く

![debug1](/images/20260502_view-transition-table/debug1.png)

### 2. Animationを停止/スロー状態にする

![debug2](/images/20260502_view-transition-table/debug2.png)

※以下で、アニメーションログをクリアできます

![debug3](/images/20260502_view-transition-table/debug3.png)

※以下で、アニメーション速度を25%にスローできます

![debug4](/images/20260502_view-transition-table/debug4.png)

### 3. View Transitionを実行する

停止状態であれば、ボタンクリック等でView Transitionがトリガーした際に、`::view-transition` が表示されたままで、アニメーション実行直後の初期状態で止まります

私がView Transitionの設定をミスった時は、複数のView Transitionが表示されていたり、view-transition-nameが重複していたりしました

以下のようにピンク色の `::view-transition` が見えればOKです

![debug5](/images/20260502_view-transition-table/debug5.png)


## おわりに

`::view-transition-group()` や `::view-transition-image-pair()` など色々あって最初は混乱しました

ですが、mozillaのドキュメントを見て分かるとおり、`inherit` で親要素のアニメーション設定を継承しているだけでした...

最初のうちはデフォルトのアニメーション `animation-fill-mode: both` を無効化して、`::view-transition-old()` や `::view-transition-new()` にそれぞれアニメーションを定義すると理解が深まりやすいかもしれません

P.S. 深夜に書いているので分かりづらかったらすみません...

## 補足・参考

最新のView Transition APIについてのブログ
@[card](https://developer.chrome.com/docs/css-ui/view-transitions/element-scoped-view-transitions?hl=ja)
デフォルトでView Transition APIに適用されているアニメーションについて
@[card](https://developer.mozilla.org/ja/docs/Web/CSS/Reference/Selectors/::view-transition-group)
View Transition APIの仕様 (文章が多いですが、途中で動画が豊富にあります!)
@[card](https://drafts.csswg.org/css-view-transitions-1/)
ReactなどバニラのHTML、TypeScriptでない場合はflushSync等を利用するよう気を付けてください
@[card](https://ja.react.dev/reference/react-dom/flushSync)
ReactではView Transition API用のコンポーネントが近々リリース予定です
@[card](https://react.dev/reference/react/ViewTransition)