---
title: "【Bedrock×Step Functions】AWS障害情報を日本語訳してメール通知"
emoji: "🦔"
type: "tech"
topics: ["aws", "bedrock", "stepfunctions", "health", "sns"]
published: true
---

## 概要

AWS Health Event(AWS障害情報)をEventBridgeで検知し、Step Functionsをトリガーして、Bedrockを呼び出し、AWS障害情報を日本語訳して、SNSでメール通知します

![](/images/20241117_aws-bedrock-stepfunctions/2.png)

## 背景

AWS Health Dashboardを見て分かるとおり、AWSの障害情報が英語になっているためです

![](/images/20241117_aws-bedrock-stepfunctions/1.png)

## 前提

Bedrockでモデルアクセスが有効になっていること
※Claude 3 Haikuを利用します

![](/images/20241117_aws-bedrock-stepfunctions/3.png)

## 構築

### SNS

トピックを `スタンダード` で作成します
![](/images/20241117_aws-bedrock-stepfunctions/4.png)
![](/images/20241117_aws-bedrock-stepfunctions/5.png)

サブスクリプションを作成します
![](/images/20241117_aws-bedrock-stepfunctions/6.png)

任意の宛先メールアドレスを入力してください
![](/images/20241117_aws-bedrock-stepfunctions/7.png)

宛先メールアドレスにサブスクリプション確認用のメールが届くため、リンクを右クリックしてコピーします
![](/images/20241117_aws-bedrock-stepfunctions/8.png)

`サブスクリプションの確認` をクリックします
![](/images/20241117_aws-bedrock-stepfunctions/9.png)

コピーしたURLを入力し、`サブスクリプションの確認` をクリックします
![](/images/20241117_aws-bedrock-stepfunctions/10.png)

サブスクリプションが `確認済み` になりました
![](/images/20241117_aws-bedrock-stepfunctions/11.png)

### Step Functions

![](/images/20241117_aws-bedrock-stepfunctions/12.png)

以下のコードをコピペします

:::message alert
SNSトピックのARNは、適宜変更してください
:::

※今回は、Claude 3 Haikuを利用しますが、言語モデルによって、Parametersの設定が異なるため注意してください

※`detail-type` のように「-」ハイフンを含む文字列を指定する場合は、`$.detail-type` ではなく、`$.["detail-type"]` にする必要がありました

```json
{
  "Comment": "Translate Health Event into Japanese and send an email notification.",
  "StartAt": "Translate Health Event",
  "States": {
    "Translate Health Event": {
      "Type": "Task",
      "Resource": "arn:aws:states:::bedrock:invokeModel",
      "Parameters": {
        "ModelId": "arn:aws:bedrock:ap-northeast-1::foundation-model/anthropic.claude-3-haiku-20240307-v1:0",
        "Body": {
          "anthropic_version": "bedrock-2023-05-31",
          "max_tokens": 4000,
          "messages": [
            {
              "role": "user",
              "content": [
                {
                  "type": "text",
                  "text.$": "States.Format('あなたは優秀な翻訳者です。以下の英語の文章を適切な日本語に翻訳してください。日本語訳した内容のみ出力してください。\n\n英語:{}\n\n日本語:', $.detail.eventDescription[*].latestDescription)"
                }
              ]
            }
          ]
        },
        "ContentType": "application/json",
        "Accept": "*/*"
      },
      "ResultPath": "$.result",
      "ResultSelector": {
        "translate.$": "$.Body.content[0].text"
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "BackoffRate": 2,
          "IntervalSeconds": 5,
          "MaxAttempts": 5
        }
      ],
      "Next": "Send Email"
    },
    "Send Email": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "Message.$": "States.Format('日本語: {}\n\naccount: {}\n\nservice: {}\n\ntime: {}\n\nregion: {}\n\nresources: {}\n\neventScopeCode: {}\n\n英語: {}', $.result.translate, $.account, $.detail.service, $.time, $.region, $.resources, $.detail.eventScopeCode, $.detail.eventDescription[*].latestDescription)",
        "TopicArn": "SNSトピックのARN"
      },
      "End": true
    }
  }
}
```

`コード` からコピペします
![](/images/20241117_aws-bedrock-stepfunctions/13.png)

`設定` から `ステートマシン名` を入力して `作成` をクリックします
![](/images/20241117_aws-bedrock-stepfunctions/14.png)

### EventBridge

`ルール` を作成します

![](/images/20241117_aws-bedrock-stepfunctions/15.png)
![](/images/20241117_aws-bedrock-stepfunctions/16.png)

イベントパターン

```json
{
  "source": ["aws.health"]
}
```

![](/images/20241117_aws-bedrock-stepfunctions/17.png)
![](/images/20241117_aws-bedrock-stepfunctions/18.png)
![](/images/20241117_aws-bedrock-stepfunctions/19.png)

あとは、Health Event(AWS障害情報)が発生するのを待つだけです…

## テスト

Eventを手動で発生させて、メールが届くか確認します

### イベントパターンの変更

テストするために一時的に `test` に変更します

```json
{
  "source": ["test"]
}
```

![](/images/20241117_aws-bedrock-stepfunctions/20.png)

### テスト用イベントの送信

`イベントバス` から `イベントの送信` をクリックします
![](/images/20241117_aws-bedrock-stepfunctions/21.png)

| 項目 | 値 |
| --- | --- |
| イベントバス | default |
| イベントソース(source) | test |
| 詳細タイプ(detail-type) | AWS Health Event |

`イベントの詳細(detail)` に以下のJSONを入力します

```json
{
  "eventArn": "arn:aws:health:ap-southeast-2::event/AWS_ELASTICLOADBALANCING_API_ISSUE_90353408594353980",
  "service": "ELASTICLOADBALANCING",
  "eventTypeCode": "AWS_ELASTICLOADBALANCING_OPERATIONAL_ISSUE",
  "eventTypeCategory": "issue",
  "eventScopeCode": "PUBLIC",
  "communicationId": "4826e1b01e4eed2b0f117c54306d907c713586799d76d487c9132a40149ac107-1",
  "startTime": "Thu, 26 Jan 2023 13:19:03 GMT",
  "endTime": "Thu, 26 Jan 2023 13:44:13 GMT",
  "lastUpdatedTime": "Thu, 26 Jan 2023 13:44:13 GMT",
  "statusCode": "open",
  "eventRegion": "ap-southeast-2",
  "eventDescription": [{
    "language": "en_US",
    "latestDescription": "Actions speak louder than words"
  }],
  "affectedAccount": "123456789012",
  "page": "1",
  "totalPages": "1"
}
```

![](/images/20241117_aws-bedrock-stepfunctions/22.png)
![](/images/20241117_aws-bedrock-stepfunctions/23.png)
![](/images/20241117_aws-bedrock-stepfunctions/24.png)

※テストが完了したら、イベントパターンを元に戻すことを忘れないでください

## 最後に

EventBridgeやCloudWatchで監視設定をたくさん入れているのですが、生成AI(Step Functions + Bedrock)でひと手間加えて通知することが、今後の主流になりそうです

次回は、GuardDuty + Bedrock(Knowledge Base)で、GuardDutyの通知内容をカスタムして、メール通知してみたいと思います
