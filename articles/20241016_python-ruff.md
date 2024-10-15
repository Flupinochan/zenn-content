---
title: "【Ruff×VSCode】PythonでLinterとFormatterを設定"
emoji: "🐷"
type: "tech"
topics: ["python", "ruff", "vscode", "linter", "formatter"]
published: true
---

## Linter と Formatter とは

- **Linter** : コードの文法や誤りを警告してくれるツール
  例: 型定義、docstringの不足
- **Formatter** : コードの見た目を整えて、統一されたスタイルにするツール
  例: インデント、スペースの修正、import 文のソート
<br>
## Ruff とは

Python の `Linter` と `Formatter` を兼ね備えたツール
https://docs.astral.sh/ruff/
https://marketplace.visualstudio.com/items?itemName=charliermarsh.ruff

<br>
## VSCode で Ruff を拡張機能として導入する手順

### 1. VSCode をインストール
割愛します
https://code.visualstudio.com/download
<br>

### 2. テスト用に適当なフォルダを開く
※ここでは、`Ruff` という名前のフォルダを開きます
![](/images/20241016_python-ruff/1.png)
![](/images/20241016_python-ruff/2.png)
![](/images/20241016_python-ruff/3.png)
<br>

### 3. Profile を作成
VSCodeの設定優先度については、以下をご覧ください
https://qiita.com/tatsuyayamakawa/items/df7e5b1b0d7c336af124
![](/images/20241016_python-ruff/6.png)
`python-ruff` という名前でProfileを作成します
![](/images/20241016_python-ruff/7.png)
作成したProfileにチェックを入れてActiveにします
![](/images/20241016_python-ruff/8.png)
<br>

### 4. 拡張機能をインストール
以下2つをインストールします
- Python
- Ruff
![](/images/20241016_python-ruff/4.png)
![](/images/20241016_python-ruff/5.png)
以下の4つがインストールされます
※拡張機能の検索欄に何も入力していなければ、インストール済みの拡張機能が確認できます
![](/images/20241016_python-ruff/9.png)
<br>

### 5. VSCode Settings.json を設定
![](/images/20241016_python-ruff/10.png)
![](/images/20241016_python-ruff/11.png)
![](/images/20241016_python-ruff/12.png)

以下をコピペして保存します
```json: settings.json
{
    // .pyファイルの設定
    "[python]": {
        "editor.formatOnSave": true, // ファイル保存時に適用
        "editor.codeActionsOnSave": {
            "source.fixAll": "explicit", // 自動修正
            "source.organizeImports": "explicit" // import文のソート
        },
        "editor.defaultFormatter": "charliermarsh.ruff" // フォーマッターをRuffに設定
    },
    // デフォルト設定
    "files.trimTrailingWhitespace": true, // 行末のスペースを削除
    "editor.renderWhitespace": "all", // スペースの可視化
    "editor.cursorStyle": "line-thin", // カーソルを極細スタイル
    "editor.tabSize": 4, // タブを4スペースに設定
    "files.eol": "\n", // ファイルの改行をLFに設定
    "editor.minimap.showSlider": "always", // ミニマップを常に表示
    "editor.minimap.renderCharacters": false, // ミニマップで文字を非表示
    "editor.fontSize": 17, // エディタのフォントサイズ
    "terminal.integrated.fontSize": 17, // ターミナルのフォントサイズ
    "debug.console.fontSize": 17, // デバッグコンソールのフォントサイズ
    "markdown.preview.fontSize": 17, // Markdownプレビューのフォントサイズ
    "chat.editor.fontSize": 17, // チャットエディタのフォントサイズ
    "scm.inputFontSize": 17, // SCM（ソース管理）の入力フォントサイズ
    "git.autofetch": true, // Gitの自動フェッチを有効にする
    // "ruff.configuration": "pyproject.toml", // Ruff設定ファイルのパス
    "ruff.format.preview": false, // プレビュー中の機能を無効化
    "ruff.lint.select": ["ALL"], // 全てのLinter機能を有効化
    "ruff.lint.extendSelect": ["ALL"], // 全てのLinter拡張機能を有効化
    "ruff.lint.ignore": [
        "T201", // print() の使用を許可
        "D400", "D415", // docstringの末尾がピリオド.の強制を無効化
        "INP001", // __init__.py の設定不要
        "ERA001", // コメントアウトされたコードをOK
        "RET504", // 直接returnしなくてもOK
    ] // Linter・Formatter無効化設定
}
```

![](/images/20241016_python-ruff/13.png)

<br>

### 6. Linter・Formatter をテスト
`test.py` という名前で、以下のpythonファイルを作成します
![](/images/20241016_python-ruff/14.png)

**■修正前**
```python: test.py
# 1. 何のコードファイルかを示すdocstringがファイルの最初にない
# 2. ソートされておらず、不要なimport文がある
import os
import sys
from datetime import datetime
import random
import math
import json

my_list=[1,2,3,4,5] # 3. スペースのないリスト

# 4. アノテーション(型)定義をしていない
def example_function(x, y):
  # 5. 関数の説明がない
    random_value = random.randint(1, 10)
    sum_of_list = sum(my_list)
    result = x + y + sum_of_list + math.sqrt(random_value)
    return result
```
![](/images/20241016_python-ruff/15.png)

保存すると、以下のようにFormatterが適用されます
![](/images/20241016_python-ruff/16.png)

しかし、まだ黄色い波線で警告が表示されているため、手動で修正していきます
「波線にカーソルを合わせる」と原因が表示されます
詳細な原因を確認するには、「青色のURL」をクリックします
![](/images/20241016_python-ruff/17.png)
![](/images/20241016_python-ruff/18.png)

他も確認して修正すると、以下のようになりました

![](/images/20241016_python-ruff/19.png)

`result` の波線は、「`result` 変数に代入せずに直接returnしなさい」という警告です
しかし、ここでは、一度変数に代入してからreturnしたいため、警告を無効化する設定をします
以下の画像のように、`settings.json` で `"ruff.lint.ignore"` に、無効化したいルールのIDを入れてください
![](/images/20241016_python-ruff/20.png)

`settings.json` を保存すると、以下のように警告が消えます
![](/images/20241016_python-ruff/21.png)


**■修正後**
```python: test.py
"""Ruffテスト用コード"""

import math
import secrets

my_list = [1, 2, 3, 4, 5]


def example_function(x: int, y: int) -> float:
    """適当な数値を返す関数"""
    random_value = secrets.randbelow(10) + 1
    sum_of_list = sum(my_list)
    result = x + y + sum_of_list + math.sqrt(random_value)
    return result

```