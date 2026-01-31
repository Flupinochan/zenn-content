---
title: "x-ray-sdkのサポート終了に伴うOpenTelemetryへの移行"
emoji: "🌵"
type: "tech"
topics: ["xray", "opentelemetry", "powertools", "lambda", "ecs"]
published: true
---

## はじめに

AWS X-Ray SDK for Pythonが2026年2月25日でメンテナンスモードに移行し、2027年にサポート終了予定です

OpenTelemetryへの移行が推奨されています

早くも来月にメンテナンスモードになるため、急いでOpenTelemetryの使い方を勉強しました!

@[card](https://aws.amazon.com/jp/blogs/news/announcing-aws-x-ray-sdks-daemon-end-of-support-and-opentelemetry-migration/)

また、Powertools for AWS Lambda (以下、Powertools) を利用している方も注意が必要です

Powertoolsのトレース機能は内部的にX-Ray SDKを利用しているため、将来的な影響が懸念されます

@[card](https://docs.aws.amazon.com/powertools/python/latest/core/tracer/)

:::message
X-Rayサービスが終了するわけではありません
X-Rayというトレースの使い方がOpenTelemetryという標準化された方法に変わるだけです
:::

## 対象者

- AWS Lambda または ECS で X-Ray を利用している方
- OpenTelemetry への移行を検討している方

**記事内のコードサンプルについて:**

- Python で記載されています
- AWS CDK (TypeScript) でインフラ構築をしています

**特に以下の方におすすめ:**

- Powertools for AWS Lambda を利用している方

## 結論

### X-Ray SDK (トレースのみ) を利用している場合 → 移行

OpenTelemetryへの移行を推奨します

**理由:**

- Lambdaは公式Layerが提供されており、設定が容易
- ECSはサイドカーコンテナの設定が必要だが、X-Ray SDKの設定と大差ない

### Powertools (ロガー + トレース) を利用している場合 → 様子見

現時点ではPowertoolsの継続利用を推奨します

**理由:**

- [公式ロードマップ](https://docs.aws.amazon.com/powertools/python/3.24.0/roadmap/) に `Add support for OpenTelemetry in our tracer utility` が記載
- 利用者側が手動で移行せずとも、Powertoolsのアップデートで対応できる可能性がある

**補足:**

また、OpenTelemetryのログ機能はまだ実用段階ではなく (後述の検証結果を参照)、現時点でログとトレースを併用すると管理が煩雑になります

私は、Powertoolsでロガー、トレースの実装をし続けようと思います

## Lambdaでの実装

サンプルコードは以下になります

@[card](https://github.com/Flupinochan/aws-lambda-ecs-open-telemetry-sample)

### パターン1 AWSOpenTelemetryDistroPython

`AWSOpenTelemetryDistroPython` は公式のLambda Layerで追加するだけでOpenTelemetryが利用可能になります

#### レイヤーを追加

![1.png](/images/20260131_aws-ecs-lambda-open-telemetry/1.png)

#### 環境変数を追加

| キー                    | 値                   |
| ----------------------- | -------------------- |
| AWS_LAMBDA_EXEC_WRAPPER | /opt/otel-instrument |

![2.png](/images/20260131_aws-ecs-lambda-open-telemetry/2.png)

#### サンプルコード

```python:python
import logging

from opentelemetry import trace

logger = logging.getLogger("sample-logger")
tracer = trace.get_tracer("sample-tracer")

@tracer.start_as_current_span("step1")
def step1():
    logger.info("step1 called")

# ~~~省略~~~
```

#### ログ出力結果

Powertoolsのログと比較して結構増えます...

![5.png](/images/20260131_aws-ecs-lambda-open-telemetry/5.png)

#### トレース出力結果

![6.png](/images/20260131_aws-ecs-lambda-open-telemetry/6.png)

### パターン2 AWS CDK adotInstrumentation

AWS CDKでLambdaを定義する際、`adotInstrumentation` プロパティを使用してOpenTelemetry用のLambda Layerを追加できます

:::message
ただし、このLayerは非推奨のため注意してください

AWS CDKでOpenTelemetryを利用する場合、`adotInstrumentation`プロパティは使用せず、通常のLayer追加方法で`AWSOpenTelemetryDistroPython` (アカウントID: 615299751070) を指定することを推奨します

AWSマネジメントコンソールから選べる `AWSOpenTelemetryDistroPython` が新しいLayer ARNです
:::

| Layer名                            | アカウントID | Layer ARN                                                                           |
| ---------------------------------- | ------------ | ----------------------------------------------------------------------------------- |
| 推奨: AWSOpenTelemetryDistroPython | 615299751070 | arn:aws:lambda:ap-northeast-1:615299751070:layer:AWSOpenTelemetryDistroPython:20    |
| 非推奨: aws-otel-python            | 901920570463 | arn:aws:lambda:ap-northeast-1:901920570463:layer:aws-otel-python-amd64-ver-1-29-0:1 |

@[card](https://aws-otel.github.io/docs/getting-started/lambda#not-recommended-using-the-legacy-adot-lambda-layers-with-embedded-collector)

#### CDKでLambda関数を作成

```typescript:typescript
const fn = new lambda.Function(this, "MyFunction", {
    functionName: `${cdk.Stack.of(this).stackName}-function`,
    runtime: lambda.Runtime.PYTHON_3_13,
    handler: "app.lambda_handler",
    code: lambda.Code.fromAsset(
        path.join(__dirname, "..", "..", "services", "otel-lambda", "src"),
    ),
    // 以下で追加 (非推奨)
    adotInstrumentation: {
        layerVersion: lambda.AdotLayerVersion.fromPythonSdkLayerVersion(
            lambda.AdotLambdaLayerPythonSdkVersion.LATEST,
        ),
        execWrapper: lambda.AdotLambdaExecWrapper.INSTRUMENT_HANDLER,
    },
    architecture: lambda.Architecture.X86_64,
    timeout: cdk.Duration.seconds(30),
    memorySize: 256,
    tracing: lambda.Tracing.ACTIVE,
    loggingFormat: lambda.LoggingFormat.JSON,
    role: myRole,
    logGroup: logGroup,
    environment: {
    AWS_LAMBDA_EXEC_WRAPPER: "/opt/otel-instrument",
    },
});
```

#### ログ出力結果

個人的にはログの量がちょうどよく、使いたいところですが非推奨です

![3.png](/images/20260131_aws-ecs-lambda-open-telemetry/3.png)

#### トレース出力結果

![4.png](/images/20260131_aws-ecs-lambda-open-telemetry/4.png)

### パターン3 AWSOpenTelemetryDistroPython & AWSLambdaPowertoolsPython

- **ログ**: Powertools
- **トレース**: OpenTelemetry

併用するパターンです

邪道な気もしますが、Powertoolsの**ロガーはサポートが終了するわけではなく**、OpenTelemetryよりも機能が豊富で柔軟なためです

:::message alert
設定なしで併用するとログが重複して出力されます
後述のコード例にある `logging.getLogger(logger.service).propagate = False` の設定が必須です
:::

#### レイヤーを追加

| レイヤー名                   |
| ---------------------------- |
| AWSOpenTelemetryDistroPython |
| AWSLambdaPowertoolsPython    |

![8.png](/images/20260131_aws-ecs-lambda-open-telemetry/8.png)

#### 環境変数を追加

| キー                                             | 値                                  |
| ------------------------------------------------ | ----------------------------------- |
| AWS_LAMBDA_EXEC_WRAPPER                          | /opt/otel-instrument                |
| OTEL_LOGS_EXPORTER                               | none                                |
| OTEL_METRICS_EXPORTER                            | none                                |
| OTEL_PROPAGATORS                                 | xray                                |
| OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED | false                               |
| POWERTOOLS_LOGGER_LOG_EVENT                      | True                                |
| POWERTOOLS_LOG_LEVEL                             | INFO                                |
| POWERTOOLS_METRICS_NAMESPACE                     | PowertoolsOtelLambdaSample          |
| POWERTOOLS_SERVICE_NAME                          | PowertoolsOtelLambdaSample-function |
| POWERTOOLS_TRACER_CAPTURE_ERROR                  | False                               |
| POWERTOOLS_TRACER_CAPTURE_RESPONSE               | False                               |
| POWERTOOLS_TRACE_DISABLED                        | True                                |
| TZ                                               | Asia/Tokyo                          |

![9.png](/images/20260131_aws-ecs-lambda-open-telemetry/9.png)

#### サンプルコード (ログ重複回避設定)

```python:python
import logging

from aws_lambda_powertools import Logger
from aws_lambda_powertools.logging.formatter import LambdaPowertoolsFormatter
from opentelemetry import trace

# logger (Powertools)
formatter = LambdaPowertoolsFormatter(
    log_record_order=["message", "level", "timestamp", "location"],
)
logger = Logger(logger_formatter=formatter)
## propagateを無効化
logging.getLogger(logger.service).propagate = False

# tracer (OpenTelemetry)
tracer = trace.get_tracer("sample-tracer")

@tracer.start_as_current_span("step1")
def step1():
    logger.info("step1 called")
```

#### ログ出力結果

![7.png](/images/20260131_aws-ecs-lambda-open-telemetry/7.png)

#### トレース出力結果

![10.png](/images/20260131_aws-ecs-lambda-open-telemetry/10.png)

## ECSでの実装 OpenTelemetry SDK

### ECS Taskの設定

#### サイドカーコンテナの設定

X-Ray SDKとは異なるコンテナイメージを使用します

| SDK               | コンテナイメージ                                                                                     |
| ----------------- | ---------------------------------------------------------------------------------------------------- |
| X-Ray SDK         | [xray/aws-xray-daemon](https://gallery.ecr.aws/xray/aws-xray-daemon)                                 |
| OpenTelemetry SDK | [aws-observability/aws-otel-collector](https://gallery.ecr.aws/aws-observability/aws-otel-collector) |

**参考:** [ECSタスク定義例](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/trace-data-containerdefinitions.html)

:::message alert
ECSタスク定義で、以下のconfigファイルを指定してください

```bash
--config=/etc/ecs/ecs-default-config.yaml
```

このファイルは**X-RayとMetrics両方に対応**しています
公式ドキュメントに記載されているconfigファイルではMetricsしか利用できないため注意してください
:::

独自のconfigファイルを作成することもできますが、[標準のconfigファイル](https://github.com/aws-observability/aws-otel-collector/tree/main/config/ecs)を使用する方が簡単で確実です

#### TaskRoleにx-rayを利用するためのIAM Policy付与

x-ray-sdkの時から変わりないと思います
以下があれば機能しました

- AWSXRayDaemonWriteAccess
- CloudWatchFullAccessV2

厳密には [IAM権限](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/trace-data.html) になると思います

### 標準ログ出力 (ConsoleLogRecordExporter) での問題

#### サンプルコード

設定の詳細は後半のコードに記載します

```python:python
import logging
import time

from opentelemetry import propagate, trace
from opentelemetry._logs import set_logger_provider
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.propagators.aws import AwsXRayPropagator
from opentelemetry.sdk._logs import (
    LoggerProvider,
    LoggingHandler,
)
from opentelemetry.sdk._logs.export import (  # ConsoleLogExporter on versions earlier than 1.39.0
    BatchLogRecordProcessor,
    ConsoleLogRecordExporter,
)
from opentelemetry.sdk.extension.aws.resource.ecs import (
    AwsEcsResourceDetector,
)
from opentelemetry.sdk.extension.aws.trace import AwsXRayIdGenerator
from opentelemetry.sdk.resources import Resource, get_aggregated_resources
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import (
    BatchSpanProcessor,
)

# tracer
ecs_resource = get_aggregated_resources([AwsEcsResourceDetector()])
service_resource = Resource.create(
    {
        "service.name": "ecs-sample-service",
        "service.namespace": "sample-namespace",
    },
)
merged_resource = ecs_resource.merge(service_resource)
provider = TracerProvider(
    id_generator=AwsXRayIdGenerator(),
    resource=merged_resource,
)
otlp_exporter = OTLPSpanExporter(endpoint="http://localhost:4318/v1/traces")
span_processor = BatchSpanProcessor(otlp_exporter)
provider.add_span_processor(span_processor)
trace.set_tracer_provider(provider)
propagate.set_global_textmap(AwsXRayPropagator())
tracer = trace.get_tracer("my.tracer.name")

# logger
provider = LoggerProvider(resource=service_resource)
processor = BatchLogRecordProcessor(ConsoleLogRecordExporter())
provider.add_log_record_processor(processor)
set_logger_provider(provider)
handler = LoggingHandler(level=logging.INFO, logger_provider=provider)
handler.setFormatter(logging.Formatter("%(message)s"))
logging.basicConfig(handlers=[handler], level=logging.INFO)
logger = logging.getLogger("my.logger.name")


@tracer.start_as_current_span("step1")
def step1() -> dict[str, object]:
    """Step 1: Simulate I/O-like work."""
    time.sleep(0.25)
    logger.info("step1 called")
    return {"value": 123}


@tracer.start_as_current_span("step2")
def step2() -> dict[str, object]:
    """Step 2: Simulate CPU-like work."""
    data = step1()
    time.sleep(0.15)
    logger.info("step2 called")
    return {"value": data.get("value"), "transformed": True}


@tracer.start_as_current_span("step3")
def step3() -> None:
    """Step 3: Simulate a downstream call."""
    time.sleep(0.35)
    logger.info("step3 called")


@tracer.start_as_current_span("do_work")
def run_pipeline() -> None:
    """Run the step pipeline with nested calls."""
    step2()
    step3()


@tracer.start_as_current_span("main")
def main():
    """Main function."""
    trace_id = trace.get_current_span().get_span_context().trace_id
    logger.info(f"Trace ID: {trace_id:032x}")
    run_pipeline()


if __name__ == "__main__":
    main()
```

#### ログ出力結果

![11.png](/images/20260131_aws-ecs-lambda-open-telemetry/11.png)

お気づきになられたろうか...
OpenTelemetryの標準出力ログは、JSON形式ではあるのですが、ログレコード (キー: 値) ごとに改行があるため、分割されてしまっています

:::details 理由

以下のブログに記載されています

@[card](https://dev.classmethod.jp/articles/aws-lambda-json-output/)

以下がOpenTelemetryのログ構造で `json.dumps({}, indent=indent)` が原因です

```python:pyhon
@dataclass(frozen=True)
class ReadableLogRecord:
    """Readable LogRecord should be kept exactly in-sync with ReadWriteLogRecord, only difference is the frozen=True param."""

    log_record: LogRecord
    resource: Resource
    instrumentation_scope: InstrumentationScope | None = None
    limits: LogRecordLimits | None = None

    @property
    def dropped_attributes(self) -> int:
        if isinstance(self.log_record.attributes, BoundedAttributes):
            return self.log_record.attributes.dropped
        return 0

    def to_json(self, indent: int | None = 4) -> str:
        return json.dumps(
            {
                "body": self.log_record.body,
                "severity_number": self.log_record.severity_number.value
                if self.log_record.severity_number is not None
                else None,
                "severity_text": self.log_record.severity_text,
                "attributes": (
                    dict(self.log_record.attributes)
                    if bool(self.log_record.attributes)
                    else None
                ),
                "dropped_attributes": self.dropped_attributes,
                "timestamp": ns_to_iso_str(self.log_record.timestamp)
                if self.log_record.timestamp is not None
                else None,
                "observed_timestamp": ns_to_iso_str(
                    self.log_record.observed_timestamp
                ),
                "trace_id": (
                    f"0x{format_trace_id(self.log_record.trace_id)}"
                    if self.log_record.trace_id is not None
                    else ""
                ),
                "span_id": (
                    f"0x{format_span_id(self.log_record.span_id)}"
                    if self.log_record.span_id is not None
                    else ""
                ),
                "trace_flags": self.log_record.trace_flags,
                "resource": json.loads(self.resource.to_json()),
                "event_name": self.log_record.event_name
                if self.log_record.event_name
                else "",
            },
            indent=indent,
            cls=BytesEncoder,
        )
```

:::

### カスタムログExporterでの解決方法

#### サンプルコード

Exporterとはテレメトリデータの**出力先**と**フォーマット**を定義するコンポーネントです

`LogRecordExporter` というベースクラスを継承してカスタムExporterを作成し、`json.dumps()` の `indent=None` を指定することで、1行にJSON形式で出力できます

```python:python
import datetime
import json
import logging
import sys
import time
from collections.abc import Sequence
from os import linesep

from opentelemetry import propagate, trace
from opentelemetry._logs import set_logger_provider
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.propagators.aws import AwsXRayPropagator
from opentelemetry.sdk._logs import (
    LoggerProvider,
    LoggingHandler,
    ReadableLogRecord,
)
from opentelemetry.sdk._logs.export import (  # ConsoleLogExporter on versions earlier than 1.39.0
    BatchLogRecordProcessor,
    LogRecordExporter,
    LogRecordExportResult,
)
from opentelemetry.sdk.extension.aws.resource.ecs import (
    AwsEcsResourceDetector,
)
from opentelemetry.sdk.extension.aws.trace import AwsXRayIdGenerator
from opentelemetry.sdk.resources import Resource, get_aggregated_resources
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import (
    BatchSpanProcessor,
)

# tracer
ecs_resource = get_aggregated_resources([AwsEcsResourceDetector()])
service_resource = Resource.create(
    {
        "service.name": "ecs-sample-service",
        "service.namespace": "sample-namespace",
    },
)
merged_resource = ecs_resource.merge(service_resource)
provider = TracerProvider(
    id_generator=AwsXRayIdGenerator(),
    resource=merged_resource,
)
## 出力先は以下のOTEL configファイルを参照 (デフォルトはlocalhost:4318)
## config/ecs/ecs-default-config.yaml
otlp_exporter = OTLPSpanExporter(endpoint="http://localhost:4318/v1/traces")
span_processor = BatchSpanProcessor(otlp_exporter)
provider.add_span_processor(span_processor)
trace.set_tracer_provider(provider)
propagate.set_global_textmap(AwsXRayPropagator())
tracer = trace.get_tracer("my.tracer.name")


# logger
class CustomLogRecordExporter(LogRecordExporter):
    """カスタムのログ出力設定"""

    def __init__(self):
        self.out = sys.stdout

        def formatter(record: ReadableLogRecord) -> str:
            rr = record.log_record

            # JST変換
            timestamp = rr.timestamp
            if rr.timestamp is not None:
                ts = datetime.datetime.fromtimestamp(
                    rr.timestamp / 1e9,
                    tz=datetime.UTC,
                )
                jst = datetime.timezone(datetime.timedelta(hours=9))
                timestamp = ts.astimezone(jst).isoformat(timespec="microseconds")

            # 出力するLogRecordの内容を定義
            obj = {
                "message": rr.body,
                "level": rr.severity_text,
                "timestamp": timestamp,
                "attributes": dict(rr.attributes) if bool(rr.attributes) else None,
                "trace_id": (
                    f"{format(rr.trace_id, '032x')}" if rr.trace_id is not None else ""
                ),
            }
            # Noneで改行をなくしてJSON出力
            return json.dumps(obj, indent=None) + linesep

        self.formatter = formatter

    def export(self, batch: Sequence[ReadableLogRecord]):
        for log_record in batch:
            self.out.write(self.formatter(log_record))
        self.out.flush()
        return LogRecordExportResult.SUCCESS

    def shutdown(self):
        pass


provider = LoggerProvider(resource=service_resource)
# 標準のログFormatであるConsoleLogRecordExporterを利用することも可能
processor = BatchLogRecordProcessor(CustomLogRecordExporter())
provider.add_log_record_processor(processor)
set_logger_provider(provider)
handler = LoggingHandler(level=logging.INFO, logger_provider=provider)
# bodyキーに出力される内容をmessageのみにする
# OpenTelemetryのloggerにはTracesとMetricsとは異なり、
# Logs API(.infoや.error)がないため普通にloggingの仕組みを使う
handler.setFormatter(logging.Formatter("%(message)s"))
logging.basicConfig(handlers=[handler], level=logging.INFO)
logger = logging.getLogger("my.logger.name")


@tracer.start_as_current_span("step1")
def step1() -> dict[str, object]:
    """Step 1: Simulate I/O-like work."""
    time.sleep(0.25)
    logger.info("step1 called")
    return {"value": 123}


@tracer.start_as_current_span("step2")
def step2() -> dict[str, object]:
    """Step 2: Simulate CPU-like work."""
    data = step1()
    time.sleep(0.15)
    logger.info("step2 called")
    return {"value": data.get("value"), "transformed": True}


@tracer.start_as_current_span("step3")
def step3() -> None:
    """Step 3: Simulate a downstream call."""
    time.sleep(0.35)
    logger.info("step3 called")


@tracer.start_as_current_span("do_work")
def run_pipeline() -> None:
    """Run the step pipeline with nested calls."""
    step2()
    step3()


@tracer.start_as_current_span("main")
def main():
    """Main function."""
    trace_id = trace.get_current_span().get_span_context().trace_id
    # X-Rayでtrace idで検索する際は接頭辞の0xは不要
    logger.info(f"Trace ID: {trace_id:032x}")
    run_pipeline()


if __name__ == "__main__":
    main()
```

#### ログ出力結果

![12.png](/images/20260131_aws-ecs-lambda-open-telemetry/12.png)

#### トレース出力結果

![13.png](/images/20260131_aws-ecs-lambda-open-telemetry/13.png)

## おわりに

OpenTelemetryはログ、トレース、メトリクスを標準化したObservabilityフレームワークですが、ログはまだ実用段階ではないと感じました

ログはPowertools、トレースはOpenTelemetryという併用も可能ですが、チームの学習コストと管理負担を考慮すると、もう少しPowertoolsだけで運用しようと思います

Powertoolsの裏側だけOpenTelemetryに移行されて使い方が変わらないことを祈っています

## 参考資料

@[card](https://aws-otel.github.io/docs/getting-started/lambda)
@[card](https://opentelemetry.io/docs/languages/python/instrumentation/)
@[card](https://opentelemetry.io/docs/specs/otel/configuration/sdk-environment-variables/)
@[card](https://docs.aws.amazon.com/ja_jp/xray/latest/devguide/migrate-xray-to-opentelemetry-python.html)
