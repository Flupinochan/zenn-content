---
title: "【Powertools】Lambdaのベストプラクティス"
emoji: "👌"
type: "tech"
topics: ["python", "AWS", "Lambda", "cloudwatch", "powertools"]
published: true
---

## 概要

- Logger、Tracer、Metricsを含む様々なツールをベストプラクティスで提供
- AWS公式Layerであり、Lambdaで簡単に使用可能

:::message alert
2024/11/16現在、AWS公式Layerは、Python 3.13には対応していないため、Python 3.12でLambdaを作成してください
:::

## マニュアル

Pythonだけでなく、TypeScript等の他の言語もサポートされています

@[card](https://docs.powertools.aws.dev/lambda/python/latest/)
@[card](https://docs.powertools.aws.dev/lambda/typescript/latest/)

## Lambda Powertools設定

### Layerの追加

![](/images/20241116_aws-power-tools/1.png)
![](/images/20241116_aws-power-tools/2.png)

### 環境変数の設定

![](/images/20241116_aws-power-tools/3.png)

| キー                               | 値                   | 説明                                       |
| ---------------------------------- | -------------------- | ------------------------------------------ |
| TZ                                 | Asia/Tokyo           | タイムゾーン                               |
| POWERTOOLS_SERVICE_NAME            | metalmental_service  | 任意のサービス名                           |
| POWERTOOLS_LOG_LEVEL               | DEBUG                | ログレベル                                 |
| POWERTOOLS_LOGGER_LOG_EVENT        | True                 | lambda_handlerのeventをログ出力する        |
| POWERTOOLS_METRICS_NAMESPACE       | AWSPowerToolsMetrics | 任意のメトリクスの名前空間                 |
| POWERTOOLS_TRACE_DISABLED          | False                | トレースの有効化                           |
| POWERTOOLS_TRACER_CAPTURE_RESPONSE | True                 | 関数のreturnをメタデータに出力する         |
| POWERTOOLS_TRACER_CAPTURE_ERROR    | True                 | 関数で発生したエラーをメタデータに出力する |

@[card](https://docs.powertools.aws.dev/lambda/python/latest/#environment-variables)

## Tracer

Lambdaのevent

```json
{
  "key1": "value1",
  "key2": "value2",
  "key3": "value3"
}
```

```python: tracer.py
"""Tracerのチュートリアル"""
# 1.Lambdaの設定でX-Rayを有効化
# 2.Lambda環境変数の設定
## POWERTOOLS_SERVICE_NAME (AnnotationにKey="Service", Value=環境変数POWERTOOLS_SERVICE_NAMEの値が追加される)
## POWERTOOLS_TRACE_DISABLED
## POWERTOOLS_TRACER_CAPTURE_RESPONSE
## POWERTOOLS_TRACER_CAPTURE_ERROR

import json

from aws_lambda_powertools import Tracer

# 3.Tracerの設定
MODULES = ["boto3", "botocore"]
tracer = Tracer(patch_modules=MODULES)


@tracer.capture_method  # 5.メソッド専用のデコレータ
def print_hello() -> str:
    """Hello, World!を出力する"""
    tracer.put_annotation(key="method_name", value="print_hello")  # 6. 注釈(Annotation)の追加
    message = "Hello, World!"
    print(message)
    return message


@tracer.capture_lambda_handler  # 4.Lambda Handler専用のデコレータ
def lambda_handler(event: dict, _context: dict) -> dict:
    """Lambdaハンドラ"""
    tracer.put_metadata(key="event", value=event)  # 7. メタデータ(Metadata)の追加
    print_hello()
    return {
        "statusCode": 200,
        "body": json.dumps({"message": "Hello, World!"}),
    }


# AnnotationとMetadataの違い
# Annotationは、検索ができるため、Key-Value形式で簡単なデータを記録する
# Metadataは、検索ができないため、ListやDictなどの複雑なデータを記録する
```

![](/images/20241116_aws-power-tools/4.png)
![](/images/20241116_aws-power-tools/5.png)

注釈(Annotation)、メタデータ、例外などで詳細を確認可能
![](/images/20241116_aws-power-tools/6.png)

ユーザー注釈(Annotation)は、メタデータと違い、検索可能
以下は、ColdStartしたLambdaを検索する例

![](/images/20241116_aws-power-tools/7.png)

## Logger

Lambdaのシステムログをjson形式にする方法については以下をご覧ください

@[card](https://zenn.dev/metalmental/articles/20251202_aws-lambda-json)

```python: loggerTest.py
"""Loggerのチュートリアル"""

# 1.Lambda環境変数の設定
## POWERTOOLS_SERVICE_NAME
## POWERTOOLS_LOG_LEVEL
## POWERTOOLS_LOGGER_LOG_EVENT
## TZ
from aws_lambda_powertools import Logger
from aws_lambda_powertools.logging.formatter import LambdaPowertoolsFormatter  # 7.フォーマット設定
from aws_lambda_powertools.utilities.typing import LambdaContext  # 2.LambdaContextの型定義

formatter = LambdaPowertoolsFormatter(log_record_order=["timestamp", "level", "location", "message"])  # 7.ログ出力順序の設定
logger = Logger(logger_formatter=formatter)


def hello_world() -> None:
    """ログ出力のテスト"""
    logger.info("Hello, World!")


def call_error() -> None:
    """エラーを発生させる関数"""
    try:
        raise ValueError("something went wrong!")
    except ValueError:
        logger.exception("Raised an exception!")  # 6.ログレベルエラーでログ出力(スタックトレース情報が含まれる)


# @logger.inject_lambda_context(log_event=True)  # ※log_event=Trueの場合、event内容がログに出力される
@logger.inject_lambda_context()  # 3.lambda_handlerに設定し、LambdaContext(Lambda関数名やMemorySize等)をロガーに入れる
def lambda_handler(_event: dict, _context: LambdaContext) -> None:
    """Lambdaハンドラ"""
    common_keys = {
        "common_key1": "common_value1",
        "common_key2": "common_value2",
    }
    logger.append_keys(**common_keys)  # 4.LambdaContextに共通でログ出力される追加のキーを追加
    hello_world()
    extra_log = {"extra_key": "extra_value"}
    logger.info("こんにちは、ワールド!", extra=extra_log)  # 5.LambdaContextに専用のキー(dict)を追加してログ出力 (他のメッセージには出力されない)
    print(logger.get_current_keys())  # 6.ログ出力時のキーを確認
    call_error()


# append_keys と thread_safe_append_keys の違い
# thread_safe_append_keys は、マルチスレッド環境で使用し、他のスレッドに影響を与えずにキーを追加する
```

messageに加え、タイムスタンプやログレベルなどの情報が自動的に出力されるため、ログの解析や追跡が容易になります

:::message
別ファイルなどでロガー設定を継承したい場合は、`logger = Logger(child=True)` とすると親ロガーの設定を継承できます
:::

![](/images/20241116_aws-power-tools/8.png)

### 単体テスト

ローカルで単体テストをする場合は、LambdaContextのLambda関数名やmemorySize等をモックする必要があります

```python: loggerTest-unittest.py
"""loggerTestのUnitテスト"""

import json
from dataclasses import dataclass

import pytest
from aws_lambda_powertools.utilities.typing import LambdaContext
from loggerTest import lambda_handler


@pytest.fixture
def lambda_context() -> LambdaContext:
    """LambdaContextのモック"""

    @dataclass
    class LambdaContext:
        function_name: str = "test"
        memory_limit_in_mb: int = 128
        invoked_function_arn: str = "arn:aws:lambda:eu-west-1:12345678910:function:test"
        aws_request_id: str = "12345678-2182-154f-163f-5f0f9a621d72"

    return LambdaContext()


def test_lambda_handler(lambda_context: LambdaContext) -> None:
    """LambdaハンドラのUnitテスト"""
    test_event = {"test": "event"}
    result = lambda_handler(test_event, lambda_context)
    assert result == {"statusCode": 200, "body": json.dumps({"message": "Hello, World!"})}
```

```bash
pytest -s loggerTest-unittest.py
```

![](/images/20241116_aws-power-tools/9.png)

## Metrics

```python: metrics.py
"""Metricsのチュートリアル"""
# Namespace
## メトリクスをグループ化する (サービス名やアプリ名)

# Dimension (ディメンション)
## 特定のメトリクスに関連付けられた固有のリソースを識別するための属性や値 (Lambda名)

# Metric
## Dimensionによって識別されたリソースに関連する何かしらの数値データで、パフォーマンスや状態を把握する (実行回数やエラー回数)

# CloudWatch Embedded Metric Format (EMF)
## CloudWatch Logsからメトリクスを生成する方法
## Powertoolsでは、メトリクスをCloudWatch Logsにログとして標準出力し、メトリクスにしている (直接CloudWatch Metricsに送信しているわけではない)
## ※特定の形式のログが出力されたらメトリクスとして集計される仕組み
## 通常のメトリクスをPutする方法と比較すると2つのメリットがある
### スロットリングされない
### コストが安い

# Lambda環境変数の設定
## POWERTOOLS_METRICS_NAMESPACE

import json
from uuid import uuid4

from aws_lambda_powertools import Metrics, single_metric
from aws_lambda_powertools.metrics import MetricResolution, MetricUnit

metrics = Metrics()  # 1.Metricsのインスタンスを作成
# metrics.set_default_dimensions(environment="dev")  # コード全体に適用されるデフォルトのディメンションを設定


def print_hello() -> None:
    """helloを出力する関数"""
    print("hello")


# @metrics.log_metrics(raise_on_empty_metrics=True) # 1つもメトリクスを送信していない場合に例外を発生させる
# @metrics.log_metrics(capture_cold_start_metric=True) # コールドスタートの場合に「ColdStart」メトリクスを送信する
# @metrics.log_metrics(default_dimensions=)
@metrics.log_metrics  # 3.Metrics送信用デコレータ(CloudWatch Logsに出力されたメトリクスのログをCloudWatch Metricsに送信する)
def lambda_handler(_event: dict, _context: dict) -> dict:
    """Lambdaハンドラ"""
    metrics.add_dimension(name="environment", value="dev")
    metrics.add_metric(name="SuccessFunctionInvocations", unit=MetricUnit.Count, value=1)  # 2.メトリクスのログ出力
    metrics.add_metric(name="SuccessFunctionInvocations2", unit=MetricUnit.Count, value=1, resolution=MetricResolution.High)  # 4.メトリクスの解像度を設定(highにするとリアルタイム性が高くなる)
    metrics.add_metadata(key="id", value=f"{uuid4()}")  # 5.メタデータの追加(CloudWatch Metricsに送信されないが、CloudWatch Logsにログとして追加で出力される)
    # コンテキストマネージャーを利用して、withブロックが終了するとメトリクスが送信される
    with single_metric(name="SuccessFunctionInvocations3", unit=MetricUnit.Count, value=1):
        metrics.add_dimension(name="environment", value="context_manager")
        print_hello()
    return {
        "statusCode": 200,
        "body": json.dumps({"message": "Hello, World!"}),
    }


# メトリクスのタイムスタンプをカスタムしたい場合
# metric.set_timestamp(timestamp)

# メトリクスの送信を手動で行いたい場合
# metrics.flush_metrics()

# EphemeralMetrics の場合は、複数のLambdaでメトリクスを共有できる
# metrics = EphemeralMetrics()
# metrics = Metrics()
```

![](/images/20241116_aws-power-tools/10.png)
![](/images/20241116_aws-power-tools/11.png)

## Event Handler(REST API)ルーティング

```python: eventHandler.py
"""REST APIのルーティング"""
# 1.API Gatewayでメソッドを作成する際に、「Proxy統合」で作成する必要がある

from __future__ import annotations

from typing import TYPE_CHECKING

from aws_lambda_powertools.event_handler import APIGatewayRestResolver, content_types
from aws_lambda_powertools.event_handler.api_gateway import CORSConfig, Response

if TYPE_CHECKING:
    from aws_lambda_powertools.event_handler.exceptions import NotFoundError
    from aws_lambda_powertools.utilities.typing.lambda_context import LambdaContext

cors_config = CORSConfig(allow_origin="*", max_age=300)
app = APIGatewayRestResolver(cors=cors_config)  # 2.CORSを有効にする
# app = APIGatewayRestResolver(cors=False) # CORSを無効にする


# 4.get
@app.get("/ping")
def get_ping() -> Response:
    """Get Ping"""
    # raise Exception("Get Ping Error") # 例外を発生させて、exception_handlerにルーティングする

    # 8.クエリ文字列の取得例
    # user_id="metalmental"&other_parameter="other"
    user_id = app.current_event.get_query_string_value(name="user_id", default_value="")  # 「user_id」というクエリパラメータを取得する
    all_query_string = app.current_event.query_string_parameters  # 全てのクエリパラメータを取得する
    print(all_query_string)

    # 9.ヘッダーの取得例
    # X-Amzn-Trace-Id:0123456789abcdef0123456789abcdef
    trace_id = app.current_event.headers.get("X-Amzn-Trace-Id")  # ヘッダーの「X-Amzn-Trace-Id」を取得する
    all_headers = app.current_event.headers  # 全てのヘッダーを取得する
    print(f"trace_id: {trace_id}")
    print(f"all_headers: {all_headers}")
    return Response(
        status_code=200,
        content_type=content_types.APPLICATION_JSON,
        body={
            "message": "Get Ping",
            "user_id": user_id,
        },
    )


# 5.post
@app.post("/ping")
def post_ping() -> Response:
    """Post Ping"""
    return Response(
        status_code=200,
        content_type=content_types.APPLICATION_JSON,
        body={"message": "Post Ping"},
    )


# 6.どのパスにもマッチしなかった場合
@app.not_found
def not_found_ping(_exc: NotFoundError) -> Response:
    """対応するパスが見つからない場合にルーティングされる"""
    return Response(
        status_code=404,
        content_type=content_types.APPLICATION_JSON,
        body={"message": "Not Found"},
    )


# 7.例外が発生した場合
@app.exception_handler(Exception)
def exception_handler_ping(exc: Exception) -> Response:
    """例外発生時にルーティングされる"""
    print(f"Exception: {exc}")
    return Response(
        status_code=500,
        content_type=content_types.APPLICATION_JSON,
        body={"message": f"Exception: {exc}"},
    )


# @app.get(".+") # 全てのGETリクエストを補足する場合
# @app.route("/todos", method=["GET", "POST"]) # 複数のメソッドを補足する場合


def lambda_handler(event: dict, context: LambdaContext) -> Response:
    """Lambda Handler"""
    return app.resolve(event, context)  # 3.ルーティングを実行する


# GraphQLのルーティングもある
# ヘッダー等のバリデーションもある
```

![](/images/20241116_aws-power-tools/12.png)
![](/images/20241116_aws-power-tools/13.png)
![](/images/20241116_aws-power-tools/14.png)
![](/images/20241116_aws-power-tools/15.png)
![](/images/20241116_aws-power-tools/16.png)
![](/images/20241116_aws-power-tools/17.png)

同様に、POSTとDELETEも作成します

![](/images/20241116_aws-power-tools/18.png)
![](/images/20241116_aws-power-tools/19.png)
![](/images/20241116_aws-power-tools/20.png)

DELETEの場合は存在しないため、not_foundにルーティングされます

![](/images/20241116_aws-power-tools/21.png)

## Validation

```python: validation.py
"""バリデーションチュートリアル"""

from aws_lambda_powertools.utilities.typing import LambdaContext
from aws_lambda_powertools.utilities.validation import SchemaValidationError, validator

# JSON Schemaの定義方法について
# https://json-schema.org/learn/getting-started-step-by-step

# 入力スキーマ(JSON Schema)
INPUT = {
    "$schema": "http://json-schema.org/draft-07/schema",
    "$id": "http://example.com/example.json",
    "type": "object",
    "title": "Sample schema",
    "description": "The root schema comprises the entire JSON document.",
    "examples": [{"user_id": "0d44b083-8206-4a3a-aa95-5d392a99be4a", "project": "powertools", "ip": "192.168.0.1"}],
    "required": ["user_id", "project", "ip"],
    "properties": {
        "user_id": {
            "$id": "#/properties/user_id",
            "type": "string",
            "title": "The user_id",
            "examples": ["0d44b083-8206-4a3a-aa95-5d392a99be4a"],
            "maxLength": 50,
        },
        "project": {
            "$id": "#/properties/project",
            "type": "string",
            "title": "The project",
            "examples": ["powertools"],
            "maxLength": 30,
        },
        "ip": {
            "$id": "#/properties/ip",
            "type": "string",
            "title": "The ip",
            "format": "ipv4",
            "examples": ["192.168.0.1"],
            "maxLength": 30,
        },
    },
}

# 出力スキーマ(JSON Schema)
OUTPUT = {
    "$schema": "http://json-schema.org/draft-07/schema",
    "$id": "http://example.com/example.json",
    "type": "object",
    "title": "Sample outgoing schema",
    "description": "The root schema comprises the entire JSON document.",
    "examples": [{"statusCode": 200, "body": {}}],
    "required": ["statusCode", "body"],
    "properties": {
        "statusCode": {
            "$id": "#/properties/statusCode",
            "type": "integer",
            "title": "The statusCode",
            "examples": [200],
            "maxLength": 3,
        },
        "body": {
            "$id": "#/properties/body",
            "type": "object",
            "title": "The body",
            "examples": [
                '{"ip": "192.168.0.1", "permissions": ["read", "write"], "user_id": "7576b683-295e-4f69-b558-70e789de1b18", "name": "Project Lambda Powertools"}',
            ],
        },
    },
}


@validator(inbound_schema=INPUT, outbound_schema=OUTPUT)
def lambda_handler(_event: dict, _context: LambdaContext) -> dict:
    """バリデーションチュートリアル"""
    try:
        # validate(event=event, schema=INPUT, envelope=envelopes.API_GATEWAY_REST) # API Gateway RESTの場合のバリデーション
        user_details = {
            "body": {
                "message": "User permissions validated.",
            },
            "statusCode": 200,
        }
    except SchemaValidationError as e:
        return {
            "body": {
                "message": f"Error: {e!s}",
            },
            "statusCode": 500,
        }
    return user_details
```

正しくValidationされるevent例

```json
{
  "user_id": "0d44b083-8206-4a3a-aa95-5d392a99be4a",
  "project": "powertools",
  "ip": "192.168.1.1"
}
```

![](/images/20241116_aws-power-tools/22.png)

Validationエラーの場合

![](/images/20241116_aws-power-tools/23.png)

## Parameter

以下のAWSサービスからパラメータを取得可能

- Systems Manager Parameter Store
- Secrets Manager
- AppConfig
- DynamoDB

Systems Manager Parameter Storeの場合

```python: parameter.py
"""パラメータ取得チュートリアル"""

# 以下4つのパラメータを簡単に取得可能にする
# Systems Manager Parameter Store
# Secrets Manager
# AppConfig
# DynamoDB

from aws_lambda_powertools.utilities import parameters
from aws_lambda_powertools.utilities.typing import LambdaContext

# from botocore.config import Config

# SSM_PROVIDER = parameters.SSMProvider(config=BOTO3_CONFIG) # configをカスタムしたい場合


def get_parameters() -> None:
    """Systems Manager Parameter Storeからパラメータを取得"""
    all_parameter = parameters.get_parameters("/powertools")  # パスによって再帰的に全てのパラメータを取得
    parameter_metalmental = parameters.get_parameter("/powertools/metalmental", force_fetch=True)  # フルパスで1つのパラメータを取得
    parameter_nekotto_chan = parameters.get_parameter("/powertools/nekotto-chan", force_fetch=True)
    for key, value in all_parameter.items():
        print(f"key: /powertools/{key}, value: {value}")
    print(f"/powertools/metalmental: {parameter_metalmental}")
    print(f"/powertools/nekotto-chan: {parameter_nekotto_chan}")


def set_parameters() -> None:
    """Systems Manager Parameter Storeにパラメータを設定"""
    parameters.set_parameter(name="/powertools/flupino-chan", value="flupino-chan", parameter_type="String", overwrite=True)


# def get_dynamodb_parameter() -> None:
#     """DynamoDBからパラメータを取得"""
#     boto3_config = Config(
#         retries={"max_attempts": 30, "mode": "standard"},
#         read_timeout=900,
#         connect_timeout=900,
#         region_name="us-west-2",
#     )
#     # key_attr: プライマリキー名、value_attr: 取得するカラム名
#     dynamodb_provider = parameters.DynamoDBProvider(
#         table_name="ChatHistory-pytzo4hizfhzfhjiq73gyqd2za-NONE",
#         key_attr="id",
#         value_attr="email",
#         config=boto3_config,
#     )
#     parameter_dynamodb = dynamodb_provider.get("b428d085-24f4-480a-ad13-b53f146b549c")
#     print(f"Email: {parameter_dynamodb}")


def lambda_handler(_event: dict, _context: LambdaContext) -> dict:
    """パラメータ取得チュートリアル"""
    get_parameters()
    set_parameters()
    # get_dynamodb_parameter()
```

以下のように、/powertools から始まる3つのパラメータを作成します

![](/images/20241116_aws-power-tools/24.png)
![](/images/20241116_aws-power-tools/25.png)
![](/images/20241116_aws-power-tools/26.png)

## Event Source Data Classes

Lambdaをトリガーにしたイベントソースのスキーマを提供

SNSでLambdaをトリガーした場合

```python: eventSourceDataClasses.py
"""SNSトリガー"""

from aws_lambda_powertools.utilities.data_classes import SNSEvent, event_source
from aws_lambda_powertools.utilities.typing import LambdaContext


# コード補完やカーソルを合わせることでサンプルイベントのドキュメントの確認が可能になる
@event_source(data_class=SNSEvent)
def lambda_handler(event: SNSEvent, _context: LambdaContext) -> None:
    """SNSからLambdaをトリガーした場合"""
    for record in event.records:
        message = record.sns.message
        subject = record.sns.subject
        print(f"subject: {subject}")
        print(f"message: {message}")
```

SNSサブスクリプションでLambdaを対象にし、以下のようにテストでメッセージを発行します

![](/images/20241116_aws-power-tools/27.png)
![](/images/20241116_aws-power-tools/28.png)

## 最後に

Loggerは、独自でライブラリを作成すると、ドキュメントやメンテナンスの管理が大変だと思います

また、Trace(X-Ray)は、`aws-xray-sdk` をpipでインストールして使用していたため、Layerを作成するのが手間でした

今後は、Powertoolsを使います
