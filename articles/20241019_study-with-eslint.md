---
title: "ESLint で学ぶ TypeScript"
emoji: "🌊"
type: "tech"
topics: ["ESLint", "TypeScript", "linter", "formatter"]
published: true
---

## 使用する ESLint/Plugin

https://eslint.org/
https://eslint.style/
https://typescript-eslint.io/
https://github.com/eslint/config-inspector
https://github.com/import-js/eslint-plugin-import
https://github.com/leo-buneev/eslint-plugin-sort-keys-fix

ルールの一覧は以下をご覧ください
[ESLint ルール](https://eslint.org/docs/latest/rules/)
[ESLint Stylistic ルール](https://eslint.style/rules)
[typescript-eslint ルール](https://typescript-eslint.io/rules/)
<br>

## 初期設定

以下でまとめてインストールします

### 以下の2ファイル `package.json` と `eslint.config.mts` を作成

```json: package.json
{
  "scripts": {
    "eslint-run": "eslint --flag unstable_ts_config \"**/*.{ts,tsx}\"",
    "eslint-fix": "eslint --flag unstable_ts_config \"/**/*.{ts,tsx}\" --fix"
  },
  "devDependencies": {
    "@eslint/js": "^9.13.0",
    "@stylistic/eslint-plugin": "^2.9.0",
    "@types/eslint__js": "^8.42.3",
    "eslint": "^9.13.0",
    "eslint-plugin-import": "^2.31.0",
    "eslint-plugin-react": "^7.37.1",
    "eslint-plugin-sort-keys-fix": "^1.1.2",
    "globals": "^15.11.0",
    "jiti": "^2.3.3",
    "typescript": "^5.6.3",
    "typescript-eslint": "^8.10.0"
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-router-dom": "^6.27.0"
  }
}
```

```typescript: eslint.config.mts
import globals from "globals";
import importPlugin from "eslint-plugin-import";
import parserTs from "@typescript-eslint/parser";
import pluginJs from "@eslint/js";
import pluginReact from "eslint-plugin-react";
import sortKeyFix from "eslint-plugin-sort-keys-fix";
import stylistic from "@stylistic/eslint-plugin";
import tseslint from "typescript-eslint";
import * as tsEslint from "@typescript-eslint/eslint-plugin";

// ファイルを指定してfixする
// flag指定でmtsファイルも対象にできる
// npx eslint .\eslint-stylistic\6.sort-imports-false.tsx --fix --flag unstable_ts_config

export default tseslint.config({
  // 設定ファイル名
  name: "my-eslint-config",
  // 適用するファイルパターン (全てのディレクトリの全ての.ts、.tsxファイルを対象)
  files: ["**/*.{ts,tsx}"],
  // 除外するファイルパターン
  ignores: ["**/node_modules/**", "**/dist/**", "**/.git/**", "**/*.{js,jsx}"],
  // プラグインの設定
  // ...pluginReact.configs.flat.recommended,
  // ...pluginReact.configs.flat['jsx-runtime'],
  plugins: {
    "@typescript-eslint": tsEslint, // ...tseslint.configs.strict でextendsする場合はコメントアウトする
    "@stylistic": stylistic,
    "import": importPlugin,
    "sort-keys-fix": sortKeyFix,
  },
  // extends: [
  //   pluginJs.configs.recommended,
  //   ...tseslint.configs.strict,
  //   ...tseslint.configs.stylistic,
  // ],
  // その他設定
  languageOptions: {
    globals: globals.browser, // Webブラウザ(Client)環境の場合
    // globals: globals.node, // Node.js(Server)環境の場合
    ecmaVersion: "latest", // ECMAScript のバージョン
    sourceType: "module", // ソースコードのタイプ(CommonJS or ESModules)
    parser: parserTs, // TypeScript用Parserを指定
    parserOptions: {
      ecmaFeatures: {
        jsx: true, // JSX(React)を使用する場合
        impliedStrict: true, // TypeScript Strict Modeを有効化する場合
      },
    },
  },
  // 手動設定ルール
  rules: {
    "no-cond-assign": ["error", "always"],
    "no-fallthrough": ["error", {
      commentPattern: "fall through", // fall throughコメントアウトがあれば許可
      allowEmptyCase: true,
      reportUnusedFallthroughComment: true,
    }],
    "array-callback-return": "error",
    "guard-for-in": "error",
    "no-unsafe-finally": "error",
    "camelcase": "error",
    "dot-notation": "error",
    "no-multi-str": "error",
    "prefer-destructuring": "error",
    "prefer-const": "error",
    "prefer-arrow-callback": "error",
    "no-constructor-return": "error",
    "valid-typeof": "error",
    "no-throw-literal": "error",
    "no-async-promise-executor": "error",
    "constructor-super": "error",
    "@typescript-eslint/consistent-type-definitions": ["error", "interface"],
    "@stylistic/quotes": ["error", "single"],
    "@stylistic/comma-spacing": ["error", { "before": false, "after": true }],
    "@stylistic/semi": ["error", "always"],
    "@stylistic/indent": ["error", 2],
    "@stylistic/line-comment-position": ["error", { "position": "above" }],
    "sort-vars": ["error", { "ignoreCase": true, }],
    "@stylistic/jsx-sort-props": ["error", {
      "callbacksLast": true,
      "shorthandFirst": false,
      "shorthandLast": true,
      "multiline": "ignore",
      "ignoreCase": false,
      "noSortAlphabetically": false,
      "reservedFirst": true,
      "locale": "auto"
    }],
    "import/first": "error",
    "import/newline-after-import": "error",
    "import/no-duplicates": "error",
    "import/order": ["error", {
      "groups": [
        "builtin",
        "external",
        ["internal", "parent", "sibling"],
        "index",
        "type"
      ],
      "newlines-between": "always",
      "alphabetize": {
        "order": "asc",
        "caseInsensitive": true
      },
      "warnOnUnassignedImports": true
    }],
    "sort-keys-fix/sort-keys-fix": "error",
    "@stylistic/padding-line-between-statements": ["error",
      { "blankLine": "always", "prev": "*", "next": "*" },
      { "blankLine": "never", "prev": "*", "next": "break" },
      { "blankLine": "never", "prev": "*", "next": "import" },
      { "blankLine": "never", "prev": ["const", "let", "var"], "next": ["const", "let", "var"] },
    ],
    "@stylistic/max-statements-per-line": ["error", { "max": 1 }],
    "@stylistic/no-multiple-empty-lines": ["error", {
      "max": 1,
      "maxEOF": 0,
      "maxBOF": 0
    }],
    "@stylistic/eol-last": ["error", "always"],
    "@stylistic/linebreak-style": ["error", "unix"],
    "@stylistic/jsx-pascal-case": ["error", {
      "allowAllCaps": true,
      "allowNamespace": false,
      "allowLeadingUnderscore": false,
    }],
    "@stylistic/jsx-child-element-spacing": "error",
    "@stylistic/jsx-self-closing-comp": ["error", {
      "component": true,
      "html": true
    }],
    "@stylistic/jsx-wrap-multilines": "error",
    "@stylistic/jsx-closing-tag-location": ["error", "tag-aligned"],
    "@stylistic/curly-newline": ["error", "always"],
    "@stylistic/no-trailing-spaces": ["error", {
      "skipBlankLines": false,
      "ignoreComments": false
    }],
  }
})
```

### npm install

```bash
npm install
```
<br>

### Linter/Formatter 実行方法1

`package.json` の `scripts` に記載されている以下のコマンドを実行します

チェックだけする場合

```bash
npm run eslint-run
```

チェックして自動修正までする場合

```bash
npm run eslint-fix
```
<br>

### Linter/Formatter 実行方法2

`VSCodeの拡張機能` を使用し、ファイル保存時にLinter/Formatterを適用する方法です

以下の拡張機能をinstallします
https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint
![](/images/20241019_study-with-eslint/1.png)

以下のように VSCodeの `settings.json` を設定します

```json
{
    "eslint.format.enable": true,
    "editor.renderWhitespace": "all",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
        "source.fixAll.eslint": "always"
    },
    "eslint.options": {
        "flags": [
            "unstable_ts_config"
        ]
    },
    "typescript.updateImportsOnFileMove.enabled": "always",
    "editor.defaultFormatter": null,
    "files.autoSave": "onFocusChange",
    "editor.formatOnType": true,
    "editor.tabSize": 2,
    "files.eol": "\n"
}
```
<br>

## ESLint で学ぶ TypeScript の基本

初期設定が完了したら、`test.ts` や `test.tsx` ファイルを作成し、任意のコードをお試しください

### 1. 【命名規則】camelcase

命名規則が `キャメルケース` であること

```typescript
// 誤ったコード例
const test_snake_case = 'test';
```

```typescript
// 正しいコード例
const testCamelCase = 'test';
```

### 2. 【let、const】prefer-const

再代入しない変数は、`const` で定義すること

```typescript
// 誤ったコード例
let letA = 'A';
letA = 'C';

let letB = 'B';

console.log(letA);
console.log(letB);
```

```typescript
// 正しいコード例
let letA = 'A';
letA = 'C';

const letB = 'B';

console.log(letA);
console.log(letB);
```

### 3. 【template literal】no-multi-str

複数行の文字列を定義する時は、バックスラッシュではなく、`バッククォート(テンプレートリテラル)` を使用すること

```typescript
// 誤ったコード例
const text = '複数行';
const multiStr = 'これは' + text + 'の文章\
です。';
```

```typescript
// 正しいコード例
const text = '複数行';
const multiStr = `これは${text}の文章
です。`;
```

### 4. 【array】array-callback-return

`配列のメソッド` では、必ず return すること

```typescript
// 誤ったコード例
const numbers = [1, 2, 3, 4];

const doubled = numbers.map((num) => {
  num * 2;
});

console.log(doubled);
```

```typescript
// 正しいコード例
const numbers = [1, 2, 3, 4];

const doubled = numbers.map((num) => {
  return num * 2;
});

console.log(doubled);
```

### 5. 【object literal】dot-notation

`ブラケット記法` ではなく、`ドット記法` を使用すること

```typescript
// 誤ったコード例
const obj = {
  name: 'MetalMental',
};

console.log(obj['name']);
```

```typescript
// 正しいコード例
const obj = {
  name: 'MetalMental',
};

console.log(obj.name);
```

### 6. 【destructuring】prefer-destructuring

オブジェクトや配列から複数の値を取得する際は、`ブラケット/ドット記法` ではなく `デストラクチャリング記法` を使用すること

```typescript
// 誤ったコード例
const obj = {
  x: 1,
  y: 2,
};

const x = obj.x;
const y = obj.y;
```

```typescript
// 正しいコード例
const obj = {
  x: 1,
  y: 2,
};

const { x, y } = obj;
```

### 7. 【if】no-cond-assign

if 文で `代入演算子=` と `比較演算子===` を間違えないこと

```typescript
// 誤ったコード例
let test = 'test';

if ((test = 'test')) {
  console.log(test);
}
```

```typescript
// 正しいコード例
const test = 'test';

if (test === 'test') {
  console.log(test);
}
```

### 8. 【for】guard-for-in

`プロトタイプチェーン` の仕組みによって継承されるプロパティのガードをすること

```typescript
// 誤ったコード例
const arr = [1, 2, 3];

for (const index in arr) {
  console.log(arr[index]);
}
```

```typescript
// 正しいコード例
const arr = [1, 2, 3];

for (const index in arr) {
  if (arr.hasOwnProperty(index)) {
    console.log(arr[index]);
  }
}
```

### 9. 【switch】no-fallthrough

switch 文の `各case` で `break` すること

```typescript
// 誤ったコード例
const number = Math.floor(Math.random() * 5) + 1;

switch (number) {
  case 1:
    console.log('数字は1です。');
    break;

  case 2:
    console.log('数字は2です。');

  case 3:
    console.log('数字は2または3です。');
    break;

  default:
    console.log('数字は1, 2, 3ではありません。');
}
```

```typescript
// 正しいコード例
const number = Math.floor(Math.random() * 5) + 1;

switch (number) {
  case 1:
    console.log('数字は1です。');
    break;

  case 2:
    console.log('数字は2です。');
    break;

  case 3:
    console.log('数字は2または3です。');
    break;

  default:
    console.log('数字は1, 2, 3ではありません。');
}
```

### 10. 【try-catch】no-unsafe-finally

try-catch-finally おける `finally` で return しないこと
※try-catch の return は、finally の return で上書きされるため

```typescript
// 誤ったコード例
function example() {
  try {
    return 'try block';
  } catch (e) {
    return 'catch block';
  } finally {
    return 'finally block';
  }
}

console.log(example());
```

```typescript
// 正しいコード例
function example() {
  let result;
  try {
    result = 'try block';
  } catch (e) {
    result = 'catch block';
  } finally {
    console.log('finally block');
  }
  return result;
}

console.log(example());
```

### 11. 【throw】no-throw-literal

文字列を直接 `throw` しないこと
※`スタックトレース情報` が出力されないため

```typescript
// 誤ったコード例
function riskyFunction() {
  try {
    throw 'エラーが起きました';
  } catch (error) {
    throw error;
  }
}
```

```typescript
// 正しいコード例
function riskyFunction() {
  try {
    throw new Error('エラーが起きました');
  } catch (error) {
    throw error;
  }
}
```

### 12. 【type guard】valid-typeof

誤った `型ガード` をしないこと
※`nunber` ではなく、`number`

```typescript
// 誤ったコード例
function checkType(value: unknown): void {
  if (typeof value === 'nunber') {
    console.log('文字列です');
  } else {
    console.log('文字列でないです');
  }
}

checkType('こんにちは'); // 文字列です
checkType(123); // 文字列でないです
```

```typescript
// 正しいコード例
function checkType(value: unknown): void {
  if (typeof value === 'number') {
    console.log('文字列です');
  } else {
    console.log('文字列でないです');
  }
}

checkType('こんにちは'); // 文字列です
checkType(123); // 文字列でないです
```

### 13. 【promise、async/await】no-async-promise-executor

`promise` で `async/await` を使用しないこと
※promise(then、catch) ⇔ async/await(try-catch)

```typescript
// 誤ったコード例
const myPromise = new Promise(async (resolve, reject) => {
    setTimeout(() => {
        const success = true;
        if (success) {
            resolve('成功しました！');
        } else {
            reject('失敗しました。');
        }
    }, 1000);
});

myPromise
    .then(result => console.log(result))
    .catch(error => console.error(error));
```

```typescript
// 正しいコード例1
const myPromise = new Promise((resolve, reject) => {
    setTimeout(() => {
        const success = true;
        if (success) {
            resolve('成功しました！');
        } else {
            reject('失敗しました。');
        }
    }, 1000);
});

myPromise
    .then(result => console.log(result))
    .catch(error => console.error(error));
```

```typescript
// 正しいコード例2
const myPromise = () => {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            const success = false;
            if (success) {
                resolve('成功しました！');
            } else {
                reject('失敗しました。');
            }
        }, 1000);
    });
};

const fetchData = async () => {
    try {
        const result = await myPromise();
        console.log(result);
    } catch (error) {
        console.error(error);
    }
};

fetchData();
```

### 14. 【class、function】prefer-arrow-callback、no-constructor-return

`.bind()` は使用せず、型情報が保持される `アロー関数` を使用すること
`constructor` は、値を初期化することが目的なため、`return` しないこと

```typescript
// 誤ったコード例1
class Example {
    value: number;

    constructor() {
        this.value = 42;
    }

    regularFunction() {
        setTimeout(function() {
            console.log(this.value);
        }, 1000);
    }
}

const example = new Example();
example.regularFunction(); // 結果: undefined
```

```typescript
// 誤ったコード例2
class Example {
    value: number;

    constructor() {
        this.value = 42;
    }

    regularFunction() {
        setTimeout(function() {
            console.log(this.value);
        }.bind(this), 1000);
    }
}

const example = new Example();
example.regularFunction(); // 結果: 42
```

```typescript
// 正しいコード例
class Example {
    value: number;

    constructor() {
        this.value = 42;
    }

    regularFunction() {
        setTimeout(() => {
            console.log(this.value);
        }, 1000);
    }
}

const example = new Example();
example.regularFunction(); // 結果: 42
```

### 15. 【extends】constructor-super

`クラスを継承` したら、必ず `super()` で親クラスのコンストラクタを実行して初期化すること

```typescript
// 誤ったコード例
class Animal {
  constructor(public name: string) {
    console.log(`Animal created: ${this.name}`);
  }
}

class Dog extends Animal {
  constructor(name: string) {
    console.log(`Dog created: ${this.name}`);
  }
}

const dog = new Dog('Buddy');
```

```typescript
// 正しいコード例
class Animal {
  constructor(public name: string) {
    console.log(`Animal created: ${this.name}`);
  }
}

class Dog extends Animal {
  constructor(name: string) {
    super(name);
    console.log(`Dog created: ${this.name}`);
  }
}

const dog = new Dog('Buddy');
```

### 16. 【interface、type】@typescript-eslint/consistent-type-definitions

似た機能を持つ `interface` と `type` どちらを使用するか統一すること?
※以下は、interfaceに統一した場合

```typescript
// 誤ったコード例
type Address = {
  street: string;
  city: string;
};
```

```typescript
// 正しいコード例
interface Address {
  street: string;
  city: string;
}
```
<br>

## ESLint Stylistic で学ぶ コードスタイルのベストプラクティス

### 1. 【クォート】@stylistic/quotes

`ダブルクォート` or `シングルクォート` に統一にすること?
※以下は、シングルクォートに統一した場合

```typescript
// 誤ったコード例
const message = "Hello, world!";
```

```typescript
// 正しいコード例
const message = 'Hello, world!';
```

### 2. 【スペース】@stylistic/comma-spacing

`カンマ,` 後にスペースを入れること

```typescript
// 誤ったコード例
const arr = [1,2,3];
```

```typescript
// 正しいコード例
const arr = [1, 2, 3];
```

### 3. 【セミコロン】@stylistic/semi

各ステートメントの末尾に `セミコロン;` を付けること
※TypeScriptでは、`自動セミコロン挿入 (ASI)` 機能があるため、各ステートメントの末尾にセミコロンを付けなくても動作はするが、予期しない動作を防ぐためにセミコロンは付けるようにする

```typescript
// 誤ったコード例
const name = 'ESLint'
```

```typescript
// 正しいコード例
const name = 'ESLint';
```

### 4. 【インデント】@stylistic/indent

インデントのスペースを統一すること
※以下は、2スペースに統一した場合

```typescript
// 誤ったコード例
const person = {
    name: 'John',
    age: 30,
    address: {
        street: '123 Main St',
        city: 'New York',
        zip: '10001'
    },
    hobbies: ['reading', 'travelling']
};
```

```typescript
// 正しいコード例
const person = {
  name: 'John',
  age: 30,
  address: {
    street: '123 Main St',
    city: 'New York',
    zip: '10001'
  },
  hobbies: ['reading', 'travelling']
};
```

### 5. 【コメント】@stylistic/line-comment-position

コメントアウトは、`横` ではなく `上` に記載すること

```typescript
// 誤ったコード例
const myName = 'MetalMental'; // 私の名前を定義する
```

```typescript
// 正しいコード例
// 私の名前を定義する
const myName = 'MetalMental';
```

### 6. 【ソート】eslint-plugin-import、eslint-plugin-sort-keys-fix、@stylistic/jsx-sort-props

`import` `objectのkey` `jsxのprops` をアルファベット順にソートすること
※`sort-imports` と `sort-keys` というルールがESLint本体にありますが、`自動fix機能がない` ため、プラグインを使用しています

```typescript
// 誤ったコード例
import { useState } from 'react';
import React from 'react';
import { Link } from 'react-router-dom';

const App = () => {
  const unsortedConfig = {
    name: 'John',
    course: 'Mathmatics',
    isStudent: true,
    age: 25,
  };

  const [count, setCount] = useState(0);

  return (
    <div>
      <button
        data-isStudent={unsortedConfig.isStudent}
        data-course={unsortedConfig.course}
        onClick={() => setCount(count + 1)}
        data-age={unsortedConfig.age}
        data-name={unsortedConfig.name}
      >
        Click me: {count}
      </button>
      <Link to="/home">Go Home</Link>
    </div>
  );
};

export default App;
```

```typescript
// 正しいコード例
import React, { useState } from 'react';
import { Link } from 'react-router-dom';

const App = () => {
  const unsortedConfig = {
    age: 25,
    course: 'Mathmatics',
    isStudent: true,
    name: 'John',
  };

  const [count, setCount] = useState(0);

  return (
    <div>
      <button
        data-age={unsortedConfig.age}
        data-course={unsortedConfig.course}
        data-isStudent={unsortedConfig.isStudent}
        data-name={unsortedConfig.name}
        onClick={() => setCount(count + 1)}
      >
        Click me: {count}
      </button>
      <Link to="/home">Go Home</Link>
    </div>
  );
};

export default App;
```

### 7. 【ファイル】@stylistic/eol-last、@stylistic/linebreak-style

ファイルの改行コードを `LF` にすること
ファイルの末尾が `改行` で終わること

### 8. 【改行】@stylistic/curly-newline

`ブロックステートメント{}`は、改行すること

```typescript
// 誤ったコード例
if (true) { console.log('true'); }
```

```typescript
// 正しいコード例
if (true) {
  console.log('true');
}
```

### 9. 【空行】@stylistic/padding-line-between-statements、@stylistic/max-statements-per-line、@stylistic/no-multiple-empty-lines

以下を制限、統一すること

- 各ステートメント前後の空行
- 連続してよい空行の数
- 1行におけるステートメントの数

```typescript
// 誤ったコード例
import React, { useState } from 'react';

import { Link } from 'react-router-dom';

const x = 5;const y = 6;const z = 7;

function foo() {
  const b = 2;

  const c = 3;
}



function bar() {
  return 8;
}
function hoge() {
  return 9;
}
```

```typescript
// 正しいコード例
import React, { useState } from 'react';
import { Link } from 'react-router-dom';

const x = 5;
const y = 6;
const z = 7;

function foo() {
  const b = 2;
  const c = 3;
}

function bar() {
  return 8;
}

function hoge() {
  return 9;
}
```

### 10. 【JSX】@stylistic/jsx-pascal-case、@stylistic/jsx-child-element-spacing、@stylistic/jsx-self-closing-comp、@stylistic/jsx-wrap-multilines、@stylistic/jsx-closing-tag-location

以下を制限、統一すること

- コンポーネント名は、大文字から始まる `パスカルケース` にする
- imgタグ等で、余分な `終了タグ` は省略する
- 複数行のJSXの場合は、`()` をつける
- インライン要素(bタグ等)とテキストが隣接する場合は、`{' '}` を使用する
- 開始タグと終了タグを整列させる

※パスカルケースの理由は、`小文字のHTMLタグ` と `大文字のコンポーネント` で区別するため

```typescript
// 誤ったコード例
import React from 'react';

const _exampleComponent: React.FC = () => {
  return <div>
    <img src="image.jpg"></img>
    <b>This text</b>
    is bold
  </div>;
};

const HelloWorld: React.FC = () => {
  return (
    <div>
      <_exampleComponent />
    </div>
  );
};
```

```typescript
// 正しいコード例
import React from 'react';

const ExampleComponent: React.FC = () => {
  return (
    <div>
      <img src="image.jpg" />
      <b>This text</b>{' '}
      is bold
    </div>
  );
};

const HelloWorld: React.FC = () => {
  return (
    <div>
      <ExampleComponent />
    </div>
  );
};
```
