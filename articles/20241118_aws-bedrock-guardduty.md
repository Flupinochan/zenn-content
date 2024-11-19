---
title: ""
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---


あなたはAWS GuardDutyの検知結果を分析するエージェントです。
以下に検索結果を英語で提供します。
ユーザーがAWS GuardDutyの検知結果を提供するので、検索結果の情報のみを使用して、対応方法が分かるように日本語に翻訳したうえで、原因と対応方法を簡潔に回答してください。
もし検索結果に対応方法が含まれていない場合は、GuardDutyの検知結果に対して正確な答えを見つけられなかったことを日本語で伝えてください。

以下は検索結果の番号付きリストです：
$search_results$

以下がGuardDutyの検知結果です:
$output_format_instructions$


★検索出来なかった場合は、以下のように固定で回答される
Sorry, I am unable to assist you with this request.

```json
{
    "Input":{
        "Text": "Backdoor:EC2/C&CActivity.Bの原因と対応方法について、日本語で簡潔に分かりやすく回答してください"
    },
    "RetrieveAndGenerateConfiguration":{
        "Type": "EXTERNAL_SOURCES",
        "ExternalSourcesConfiguration": {
            "ModelArn": "arn:aws:bedrock:us-west-2::foundation-model/anthropic.claude-3-sonnet-20240229-v1:0",
            "Sources": [
                {
                    "SourceType": "S3",
                    "S3Location": {
                        "Uri": "s3://guardduty-bedrock-knowledgebase/ec2-guardduty_compressed.pdf"
                    }
                }
            ]
        }
    }
}
```