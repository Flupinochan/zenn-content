---
title: "Lambda PythonでbasicConfigとprintログから脱却する方法"
emoji: "🦥"
type: "tech"
topics: ["python", "lambda", "powertools"]
published: true
---

## はじめに

今の職場ではバックエンドにPythonを利用しています
ログはbasicConfigやprint関数を利用しています
エラー調査がしんどいです...
ということから、簡単ですが久しぶりに記事を書くことにしました!

## Lambdaでjson形式のログを出力する方法

### 全てtext形式

デフォルトの状態です
Lambdaを作成し、print関数で標準出力します

```python
import json

def lambda_handler(event, context):
    # TODO implement
    print("hello")
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from Lambda!')
    }
```

Lambdaを実行すると以下のようなログが出力されます

![](/images/20251202_aws-lambda-json/1.png)

### システムログのみjson形式

自動で出力されるLambdaのSTARTやENDログをjson形式にします
コードの修正は不要ですが、Lambdaの設定変更が必要です

1. `設定` -> `モニタリングおよび運用ツール` -> `編集`

![](/images/20251202_aws-lambda-json/2.png)

2. `ログフォーマット` で `JSON` を選択

![](/images/20251202_aws-lambda-json/3.png)

Lambdaを実行すると以下のようなログが出力されます

![](/images/20251202_aws-lambda-json/4.png)

### 全てjson形式

AWS PowerToolsを使用します

1. コードを修正

```python
import json
from aws_lambda_powertools import Logger
from aws_lambda_powertools.logging.formatter import LambdaPowertoolsFormatter
from aws_lambda_powertools.utilities.typing import LambdaContext

formatter = LambdaPowertoolsFormatter(log_record_order=["timestamp", "level", "location", "message"])
logger = Logger(logger_formatter=formatter)

def lambda_handler(event, context):
    # TODO implement
    logger.info("hello")
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from Lambda!')
    }
```

2. `レイヤーの追加`

![](/images/20251202_aws-lambda-json/5.png)

3. `AWS レイヤー` で `AWSLambdaPowertoolsPython...` を選択

:::message
最新版のPython (現在だと3.14) は対応していないことがあるため注意してください
:::

![](/images/20251202_aws-lambda-json/6.png)

Lambdaを実行すると以下のようなログが出力されます

![](/images/20251202_aws-lambda-json/7.png)

## おわりに

AWS PowerToolsを使用すると `location` でファイル名や行数が出力されます
basicConfigやprint関数では出力されないため、エラー調査が大変です

以上、ログ出力が技術的な負債の1つとなっている現場からの報告でした...

## 参考

PowerToolsの詳細については以下をご覧ください

@[card](https://zenn.dev/metalmental/articles/20241116_aws-power-tools)
