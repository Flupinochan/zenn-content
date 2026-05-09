---
title: "AWS Step Functions個人メモ(変数とMap)"
emoji: "👋"
type: "tech"
topics: ["aws", "stepfunctions"]
published: true
---

Step Functions の入出力変数は、ややこしく忘れやすい…

生成 AI も Step Functions のコード生成に苦労している気がします…w

そのため、備忘録としてメモとサンプルのコードを記載します

## Step Functions 上の変数

$.から変数名が始まる (Linux の / のようなルートディレクトリをイメージすると分かりやすい?)

\$.Name = "MetalMental" にすれば、$.Name 変数が定義される

## InputPath と Parameters

Task への入力機能で、デフォルトでは全ての StepFunctions 上の変数が入力される

- **InputPath のみ入力する**
  - デフォルトの動作で InputPath と Parameters どちらも指定しないと、全ての StepFunctions 上の変数が Task に渡される
  - 指定しなければ、InputPath は$になり、全ての変数が入力される
  - OutputPath と同様にフィルタリングすることが目的
  - つまり、入力はデフォルトで全てのため、フィルタリングする場合のみ InputPath で特定の変数のみを指定する
- **Parameters のみ入力する**
  - Parameters のみを指定すると、Parameters のみ入力される (何らかの値をハードコーディングすることになる)
  - InputPath もデフォルトのため入力はされているが無視されている
- **InputPath と Parameters の両方を入力する**
  - InputPath と Parameters を両方指定すると Parameters が Task に渡される
  - そのうえで、Parameters に InputPath の変数を指定すると、InputPath の変数を含んだ Parameters を Task に渡すことができる
  - 用途は以下 2 つ
    - InputPath の StepFunctions 上の変数に加え、Parameters で指定したハードコーディングした値も Task に渡したい場合
    - InputPath の StepFunctions 上の変数を、Parameters で変数名を変えて Task に渡したい場合

## ResultPath と OutputPath

Task からの出力機能で、デフォルトでは Task の実行結果のみ出力され、入力は破棄される

- ResultPath は、InputPath または Task(Lambda 等)の実行結果を出力する
  - Lambda(Python)の場合は、return 句で返す
  - **InputPath のみ出力する**
    - 「null」を指定すると、InputPath のみ出力する (Task の実行結果は破棄される)
  - **Task の実行結果のみ出力する**
    - デフォルトは「$」であり、Task の実行結果のみ出力する (InputPath は破棄される)
  - **InputPath と Task の実行結果を両方出力する**
    - ResultPath に「\$.TaskResult」のように変数名を指定すると、InputPath に Task の実行結果「$.TaskResult」が追加されて両方出力される
  - InputPath の情報は出力し、後のフローでも使いまわすことが多いので、ResultPath には何らかの変数名を指定するのが良い気がする (その変数を使うかどうかはさておき)
- OutputPath は、ResultPath をフィルタリングする機能<br>
  - ResultPath の変数をフィルタリングするためであり、指定した変数のみ出力するが、あまり利用用途はない
  - 指定した変数名は、\$直下に出力されるため、後のフローで参照する際は、変数名ではなく$で参照するよう気を付ける

## Map フロー

Parallel フローと同じく並行処理するフロー

- Map
  - 配列の要素の数だけ Task を並行して実行する
  - 配列の要素全てに同じ Task を並行して実行する
  - 例 : 配列の要素が 5 つあり、Lambda を Task として実行する場合は、5 つの Lambda が並行して起動し実行される
    - ItemsPath
      - 入力する配列を指定する
    - ItemSelector
      - Task に渡す変数を指定する
        - "$$.Map.Item.Value"を指定すると、ItemsPath で指定した配列の各要素が入力される
        - 共通のパラメータを渡したい場合は、通常の Parameters と同じように、$.変数名で渡す
- Parallel
  - 事前に並行して実行する Task を定義しておく
  - 異なる Task を並行して実行する

## まとめ

- 基本的に全ての変数をデフォルトで入力/出力するため、InputPath と OutputPath は何も指定しない
- Task への入力/出力は、Parameters と ResultPath で定義する
- 直接的なループ機能はないため、以下のサンプルのようにループは実装する

## サンプル 1 InputPath を省略した場合

```python
def lambda_handler(event, context):
    name = event.get("Name")
    print(f"{name}さん、こんにちは")
```

```json
{
  "Comment": "Sample1",
  "StartAt": "List of Names",
  "States": {
    "List of Names": {
      "Type": "Pass",
      "Result": ["1山田", "2田中", "3佐々木", "4高橋", "5渡辺", "DONE"],
      "ResultPath": "$.List",
      "Next": "Loop Condition"
    },
    "Loop Condition": {
      "Type": "Choice",
      "Choices": [
        {
          "Not": {
            "Variable": "$.List[0]",
            "StringEquals": "DONE"
          },
          "Next": "Invoke Lambda"
        }
      ],
      "Default": "End"
    },
    "Invoke Lambda": {
      "Type": "Task",
      "Resource": "LambdaのARN:$LATEST",
      "TimeoutSeconds": 300,
      "HeartbeatSeconds": 60,
      "Parameters": {
        "Name.$": "$.List[0]"
      },
      "ResultPath": null,
      "Next": "Wait"
    },
    "Wait": {
      "Type": "Wait",
      "Seconds": 10,
      "Next": "Pop Element from List"
    },
    "Pop Element from List": {
      "Type": "Pass",
      "Parameters": {
        "List.$": "$.List[1:]"
      },
      "ResultPath": "$",
      "Next": "Loop Condition"
    },
    "End": {
      "Type": "Pass",
      "End": true
    }
  }
}
```

## サンプル 2 InputPath や Parameters を色々な形で指定した場合

```json
{
  "Comment": "Sample2",
  "StartAt": "List of Names",
  "States": {
    "List of Names": {
      "Type": "Pass",
      "Result": ["1山田", "2田中", "3佐々木", "4高橋", "5渡辺", "DONE"],
      "ResultPath": "$.List",
      "Next": "Loop Condition"
    },
    "Loop Condition": {
      "Type": "Choice",
      "InputPath": "$",
      "Choices": [
        {
          "Not": {
            "Variable": "$.List[0]",
            "StringEquals": "DONE"
          },
          "Next": "Invoke Lambda"
        }
      ],
      "Default": "End"
    },
    "Invoke Lambda": {
      "Type": "Task",
      "Resource": "LambdaのARN:$LATEST",
      "TimeoutSeconds": 300,
      "HeartbeatSeconds": 60,
      "InputPath": "$.List",
      "Parameters": {
        "Name.$": "$[0]"
      },
      "ResultPath": "$.TaskResult",
      "Next": "Wait"
    },
    "Wait": {
      "Type": "Wait",
      "Seconds": 10,
      "Next": "Pop Element from List"
    },
    "Pop Element from List": {
      "Type": "Pass",
      "InputPath": "$.List",
      "Parameters": {
        "List.$": "$[1:]"
      },
      "ResultPath": "$",
      "Next": "Loop Condition"
    },
    "End": {
      "Type": "Pass",
      "End": true
    }
  }
}
```

## サンプル 3 Map フロー

```python
def lambda_handler(event, context):
    name = event.get("Name")
    common_parameter = event.get("CommonParameter")
    print(f"CommonParameter: {common_parameter}")
    print(f"{name}さん、こんにちは")
```

```json
{
  "Comment": "Sample1",
  "StartAt": "List of Names",
  "States": {
    "List of Names": {
      "Type": "Pass",
      "Result": ["1山田", "2田中", "3佐々木", "4高橋", "5渡辺"],
      "ResultPath": "$.List",
      "Next": "Common Parameter"
    },
    "Common Parameter": {
      "Type": "Pass",
      "Result": "並行して処理されるTask全てに共通して渡されるパラメータ",
      "ResultPath": "$.CommonParameter",
      "Next": "Map"
    },
    "Map": {
      "Type": "Map",
      "ItemsPath": "$.List",
      "ItemSelector": {
        "Name.$": "$$.Map.Item.Value",
        "CommonParameter.$": "$.CommonParameter"
      },
      "ItemProcessor": {
        "ProcessorConfig": {
          "Mode": "INLINE"
        },
        "StartAt": "Invoke Lambda",
        "States": {
          "Invoke Lambda": {
            "Type": "Task",
            "Resource": "LambdaのARN:$LATEST",
            "TimeoutSeconds": 300,
            "HeartbeatSeconds": 60,
            "End": true
          }
        }
      },
      "Next": "End"
    },
    "End": {
      "Type": "Pass",
      "End": true
    }
  }
}
```

以下の記事が非常に参考になりました!
https://dev.classmethod.jp/articles/stepfunctions-parameters-inter-states/
