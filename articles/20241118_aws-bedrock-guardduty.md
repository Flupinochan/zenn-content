---
title: "【Bedrock×GuardDuty】脅威検出結果から日本語訳+対応方法をメール通知"
emoji: "🐬"
type: "tech"
topics: ["aws", "bedrock", "stepfunctions", "guardduty", "eventbridge"]
published: true
---

## 概要

以下のGuardDuty検出結果タイプのドキュメントをもとに、AWS GuardDutyの脅威検出結果(英語)をBedrock(RAG)で、`日本語訳` + `対応方法`をメール通知します

https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_finding-types-ec2.html

![](/images/20241118_aws-bedrock-guardduty/41.png)

※本記事で紹介する `ドキュメントとチャット(Chat with your document)` は、Embeddingをしない? ため精度が低いようですが、使用方法は他のRAGと同じで、コストが小さいため、使用しています

## 背景

- AWS GuardDutyの脅威検出結果が英語である
- 対応方法が記載されていない

![](/images/20241118_aws-bedrock-guardduty/1.png)

コンソール画面では、`情報` をクリックすると、右側に日本語訳および対応方法が表示されますが、メール通知ではその情報がありません

## 余談

https://aws.amazon.com/jp/builders-flash/202409/dify-bedrock-automate-security-operation/

AWS公式サイトで、Difyを使用した例も公開されているため、参考になさってください
ただし、EC2等のリソースはコストがかかることに注意してください

本ブログでは、マネージドサービスのみを使用するため、`コストが小さい` ことがメリットです!!

Health Eventの場合の実装例は、以下をご覧ください

https://zenn.dev/metalmental/articles/20241117_aws-bedrock-stepfunctions

## 前提

- オレゴンリージョンであること
- Bedrockでモデルアクセスが有効になっていること
※Claude 3 Sonnetを利用します

![](/images/20241117_aws-bedrock-stepfunctions/3.png)

## 構築

### GuardDuty

:::message alert
`オレゴン` リージョンであることを確認してください

2024/11/20現在は、Bedrockの機能 `ドキュメントとチャット(Chat with your document)` が東京リージョンでは対応していないためです
:::

GuardDutyを有効化します

![](/images/20241118_aws-bedrock-guardduty/2.png)
![](/images/20241118_aws-bedrock-guardduty/3.png)
![](/images/20241118_aws-bedrock-guardduty/4.png)

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
![](/images/20241117_aws-bedrock-stepfunctions/9001.png)

コピーしたURLを入力し、`サブスクリプションの確認` をクリックします
![](/images/20241117_aws-bedrock-stepfunctions/10.png)

サブスクリプションが `確認済み` になりました
![](/images/20241117_aws-bedrock-stepfunctions/11001.png)

### S3

GuardDutyの対応方法が記載されたドキュメントをS3 Bucketにアップロードします

:::message alert
Bedrock ナレッジベースの機能 `ドキュメントとチャット(Chat with your document)` の制限により、ファイルサイズはなるべく小さくする必要があります
:::

https://docs.aws.amazon.com/ja_jp/bedrock/latest/userguide/knowledge-base-chatdoc.html

以下のGuardDutyの対応方法(EC2)が記載されたドキュメントをPDFでダウンロードします
※正確性を重視して、英語のドキュメントをダウンロードします

https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_finding-types-ec2.html

上記サイトにて、`右クリック` した後、`印刷` をクリックします
※`PDF` をクリックしてもダウンロードできますが、全対応方法が記載されていてファイルサイズが大きいため、PDFファイルの内容をEC2のみに分割する必要があります
![](/images/20241118_aws-bedrock-guardduty/5.png)
![](/images/20241118_aws-bedrock-guardduty/6.png)

PDFファイルを圧縮して小さくします
※以下のサイトを利用してPDFファイルを圧縮します

https://www.ilovepdf.com/ja/compress_pdf

![](/images/20241118_aws-bedrock-guardduty/7.png)
![](/images/20241118_aws-bedrock-guardduty/8.png)
![](/images/20241118_aws-bedrock-guardduty/9.png)
![](/images/20241118_aws-bedrock-guardduty/10.png)

S3 Bucketを作成し、圧縮したPDFファイルをアップロードします

![](/images/20241118_aws-bedrock-guardduty/11.png)
![](/images/20241118_aws-bedrock-guardduty/12.png)
![](/images/20241118_aws-bedrock-guardduty/13.png)
![](/images/20241118_aws-bedrock-guardduty/14.png)
![](/images/20241118_aws-bedrock-guardduty/15.png)

`S3 URI` をコピーしておきます
![](/images/20241118_aws-bedrock-guardduty/16.png)

#### ドキュメントとチャット(Chat with your document) の動作確認

`Bedrock` に移動し、`ナレッジベース` から、`ドキュメントとチャット(Chat with your document)` を選択します
`モデル` を選択し、`最大長` を `4096` にします
![](/images/20241118_aws-bedrock-guardduty/17.png)

`チャットプロンプトのテンプレート` を編集します
![](/images/20241118_aws-bedrock-guardduty/18.png)

以下のプロンプトテンプレートを入力します

```text
あなたはAWS GuardDutyの脅威検出結果を分析するエージェントです。
AWS GuardDutyの脅威検出結果を提供するので、対応方法の番号付きリストを使用して、脅威検出結果についての対応方法が分かるように、日本語に翻訳したうえで、原因と対応方法を簡潔に回答してください。

以下は対応方法の番号付きリストです：
$search_results$

以下がAWS GuardDutyの脅威検出結果です:
$output_format_instructions$
```

`$search_results$` には、S3 BucketにアップロードしたPDFファイルの内容と、ユーザのプロンプト内容をもとに、検索結果が入ります
`$output_format_instructions$` には、ユーザのプロンプト内容が入ります

![](/images/20241118_aws-bedrock-guardduty/19.png)

`S3 の URI` を入力した後、以下のプロンプトを入力して実行します
※プロンプトテンプレート修正後に、`最大長` が `4096` になっていることを確認してください

```text
The EC2 instance i-99999999 is behaving in a manner that may indicate it is being used to perform a Denial of Service (DoS) attack using the TCP protocol.
```

![](/images/20241118_aws-bedrock-guardduty/20.png)

上記画像のように、原因と対応方法が日本語で回答されていれば、動作確認は完了です

※補足
対応するドキュメントがなく、検索できなかった場合は、以下のように回答されました

```text
Sorry, I am unable to assist you with this request.
```


### Step Functions

#### Sample Event

:::details GuardDutyのEvent例

```json
{
    "version": "0",
    "id": "70a2eca2-a15c-3d73-ce50-fe831aecbeae",
    "detail-type": "GuardDuty Finding",
    "source": "aws.guardduty",
    "account": "247574246160",
    "time": "2024-11-19T18:15:08Z",
    "region": "us-west-2",
    "resources": [],
    "detail": {
        "schemaVersion": "2.0",
        "accountId": "247574246160",
        "region": "us-west-2",
        "partition": "aws",
        "id": "9b2373d85dcf4e429cce481fa2d8a84b",
        "arn": "arn:aws:guardduty:us-west-2:247574246160:detector/eac9a2b82f6b4aa20600211ff0790627/finding/9b2373d85dcf4e429cce481fa2d8a84b",
        "type": "Backdoor:EC2/DenialOfService.Tcp",
        "resource": {
            "resourceType": "Instance",
            "instanceDetails": {
                "instanceId": "i-99999999",
                "instanceType": "m3.xlarge",
                "outpostArn": "arn:aws:outposts:us-west-2:123456789000:outpost/op-0fbc006e9abbc73c3",
                "launchTime": "2016-08-02T02:05:06.000Z",
                "platform": null,
                "productCodes": [
                    {
                        "productCodeId": "GeneratedFindingProductCodeId1",
                        "productCodeType": "GeneratedFindingProductCodeType1"
                    },
                    {
                        "productCodeId": "GeneratedFindingProductCodeId2",
                        "productCodeType": "GeneratedFindingProductCodeType2"
                    },
                    {
                        "productCodeId": "GeneratedFindingProductCodeId3",
                        "productCodeType": "GeneratedFindingProductCodeType3"
                    },
                    {
                        "productCodeId": "GeneratedFindingProductCodeId4",
                        "productCodeType": "GeneratedFindingProductCodeType4"
                    },
                    {
                        "productCodeId": "GeneratedFindingProductCodeId5",
                        "productCodeType": "GeneratedFindingProductCodeType5"
                    }
                ],
                "iamInstanceProfile": {
                    "arn": "arn:aws:iam::247574246160:example/instance/profile",
                    "id": "GeneratedFindingInstanceProfileId"
                },
                "networkInterfaces": [
                    {
                        "ipv6Addresses": [],
                        "networkInterfaceId": "eni-abcdef00",
                        "privateDnsName": "GeneratedFindingPrivateDnsName1",
                        "privateIpAddress": "10.0.0.1",
                        "privateIpAddresses": [
                            {
                                "privateDnsName": "GeneratedFindingPrivateName1",
                                "privateIpAddress": "10.0.0.1"
                            },
                            {
                                "privateDnsName": "GeneratedFindingPrivateName2",
                                "privateIpAddress": "10.0.0.2"
                            },
                            {
                                "privateDnsName": "GeneratedFindingPrivateName3",
                                "privateIpAddress": "10.0.0.3"
                            },
                            {
                                "privateDnsName": "GeneratedFindingPrivateName4",
                                "privateIpAddress": "10.0.0.4"
                            }
                        ],
                        "subnetId": "GeneratedFindingSubnetId1",
                        "vpcId": "vpc-generatedvpcid1",
                        "securityGroups": [
                            {
                                "groupName": "GeneratedFindingSecurityGroupName1",
                                "groupId": "GeneratedFindingSecurityId1"
                            },
                            {
                                "groupName": "GeneratedFindingSecurityGroupName2",
                                "groupId": "GeneratedFindingSecurityId2"
                            },
                            {
                                "groupName": "GeneratedFindingSecurityGroupName3",
                                "groupId": "GeneratedFindingSecurityId3"
                            },
                            {
                                "groupName": "GeneratedFindingSecurityGroupName4",
                                "groupId": "GeneratedFindingSecurityId4"
                            }
                        ],
                        "publicDnsName": "GeneratedFindingPublicDNSName1",
                        "publicIp": "198.51.100.1"
                    },
                    {
                        "ipv6Addresses": [],
                        "networkInterfaceId": "eni-abcdef01",
                        "privateDnsName": "GeneratedFindingPrivateDnsName2",
                        "privateIpAddress": "10.0.0.2",
                        "privateIpAddresses": [
                            {
                                "privateDnsName": "GeneratedFindingPrivateName1",
                                "privateIpAddress": "10.0.0.1"
                            },
                            {
                                "privateDnsName": "GeneratedFindingPrivateName2",
                                "privateIpAddress": "10.0.0.2"
                            },
                            {
                                "privateDnsName": "GeneratedFindingPrivateName3",
                                "privateIpAddress": "10.0.0.3"
                            },
                            {
                                "privateDnsName": "GeneratedFindingPrivateName4",
                                "privateIpAddress": "10.0.0.4"
                            }
                        ],
                        "subnetId": "GeneratedFindingSubnetId2",
                        "vpcId": "vpc-generatedvpcid2",
                        "securityGroups": [
                            {
                                "groupName": "GeneratedFindingSecurityGroupName1",
                                "groupId": "GeneratedFindingSecurityId1"
                            },
                            {
                                "groupName": "GeneratedFindingSecurityGroupName2",
                                "groupId": "GeneratedFindingSecurityId2"
                            },
                            {
                                "groupName": "GeneratedFindingSecurityGroupName3",
                                "groupId": "GeneratedFindingSecurityId3"
                            },
                            {
                                "groupName": "GeneratedFindingSecurityGroupName4",
                                "groupId": "GeneratedFindingSecurityId4"
                            }
                        ],
                        "publicDnsName": "GeneratedFindingPublicDNSName2",
                        "publicIp": "198.51.100.2"
                    },
                    {
                        "ipv6Addresses": [],
                        "networkInterfaceId": "eni-abcdef02",
                        "privateDnsName": "GeneratedFindingPrivateDnsName3",
                        "privateIpAddress": "10.0.0.3",
                        "privateIpAddresses": [
                            {
                                "privateDnsName": "GeneratedFindingPrivateName1",
                                "privateIpAddress": "10.0.0.1"
                            },
                            {
                                "privateDnsName": "GeneratedFindingPrivateName2",
                                "privateIpAddress": "10.0.0.2"
                            },
                            {
                                "privateDnsName": "GeneratedFindingPrivateName3",
                                "privateIpAddress": "10.0.0.3"
                            },
                            {
                                "privateDnsName": "GeneratedFindingPrivateName4",
                                "privateIpAddress": "10.0.0.4"
                            }
                        ],
                        "subnetId": "GeneratedFindingSubnetId3",
                        "vpcId": "vpc-generatedvpcid3",
                        "securityGroups": [
                            {
                                "groupName": "GeneratedFindingSecurityGroupName1",
                                "groupId": "GeneratedFindingSecurityId1"
                            },
                            {
                                "groupName": "GeneratedFindingSecurityGroupName2",
                                "groupId": "GeneratedFindingSecurityId2"
                            },
                            {
                                "groupName": "GeneratedFindingSecurityGroupName3",
                                "groupId": "GeneratedFindingSecurityId3"
                            },
                            {
                                "groupName": "GeneratedFindingSecurityGroupName4",
                                "groupId": "GeneratedFindingSecurityId4"
                            }
                        ],
                        "publicDnsName": "GeneratedFindingPublicDNSName3",
                        "publicIp": "198.51.100.3"
                    },
                    {
                        "ipv6Addresses": [],
                        "networkInterfaceId": "eni-abcdef03",
                        "privateDnsName": "GeneratedFindingPrivateDnsName4",
                        "privateIpAddress": "10.0.0.4",
                        "privateIpAddresses": [
                            {
                                "privateDnsName": "GeneratedFindingPrivateName1",
                                "privateIpAddress": "10.0.0.1"
                            },
                            {
                                "privateDnsName": "GeneratedFindingPrivateName2",
                                "privateIpAddress": "10.0.0.2"
                            },
                            {
                                "privateDnsName": "GeneratedFindingPrivateName3",
                                "privateIpAddress": "10.0.0.3"
                            },
                            {
                                "privateDnsName": "GeneratedFindingPrivateName4",
                                "privateIpAddress": "10.0.0.4"
                            }
                        ],
                        "subnetId": "GeneratedFindingSubnetId4",
                        "vpcId": "vpc-generatedvpcid4",
                        "securityGroups": [
                            {
                                "groupName": "GeneratedFindingSecurityGroupName1",
                                "groupId": "GeneratedFindingSecurityId1"
                            },
                            {
                                "groupName": "GeneratedFindingSecurityGroupName2",
                                "groupId": "GeneratedFindingSecurityId2"
                            },
                            {
                                "groupName": "GeneratedFindingSecurityGroupName3",
                                "groupId": "GeneratedFindingSecurityId3"
                            },
                            {
                                "groupName": "GeneratedFindingSecurityGroupName4",
                                "groupId": "GeneratedFindingSecurityId4"
                            }
                        ],
                        "publicDnsName": "GeneratedFindingPublicDNSName4",
                        "publicIp": "198.51.100.4"
                    }
                ],
                "tags": [
                    {
                        "key": "GeneratedFindingInstanceTag1",
                        "value": "GeneratedFindingInstanceValue1"
                    },
                    {
                        "key": "GeneratedFindingInstanceTag2",
                        "value": "GeneratedFindingInstanceTagValue2"
                    },
                    {
                        "key": "GeneratedFindingInstanceTag3",
                        "value": "GeneratedFindingInstanceTagValue3"
                    },
                    {
                        "key": "GeneratedFindingInstanceTag4",
                        "value": "GeneratedFindingInstanceTagValue4"
                    },
                    {
                        "key": "GeneratedFindingInstanceTag5",
                        "value": "GeneratedFindingInstanceTagValue5"
                    },
                    {
                        "key": "GeneratedFindingInstanceTag6",
                        "value": "GeneratedFindingInstanceTagValue6"
                    },
                    {
                        "key": "GeneratedFindingInstanceTag7",
                        "value": "GeneratedFindingInstanceTagValue7"
                    },
                    {
                        "key": "GeneratedFindingInstanceTag8",
                        "value": "GeneratedFindingInstanceTagValue8"
                    },
                    {
                        "key": "GeneratedFindingInstanceTag9",
                        "value": "GeneratedFindingInstanceTagValue9"
                    }
                ],
                "instanceState": "running",
                "availabilityZone": "generated-az-1a",
                "imageId": "ami-99999999",
                "imageDescription": "GeneratedFindingInstanceImageDescription"
            }
        },
        "service": {
            "serviceName": "guardduty",
            "detectorId": "eac9a2b82f6b4aa20600211ff0790627",
            "action": {
                "actionType": "NETWORK_CONNECTION",
                "networkConnectionAction": {
                    "connectionDirection": "OUTBOUND",
                    "localIpDetails": {
                        "ipAddressV4": "10.0.0.23",
                        "ipAddressV6": "1234:5678:90ab:cdef:1234:5678:90ab:cde1"
                    },
                    "remoteIpDetails": {
                        "ipAddressV4": "198.51.100.0",
                        "organization": {
                            "asn": "-1",
                            "asnOrg": "GeneratedFindingASNOrg",
                            "isp": "GeneratedFindingISP",
                            "org": "GeneratedFindingORG"
                        },
                        "country": {
                            "countryName": "GeneratedFindingCountryName"
                        },
                        "city": {
                            "cityName": "GeneratedFindingCityName"
                        },
                        "geoLocation": {
                            "lat": 0,
                            "lon": 0
                        },
                        "ipAddressV6": "1234:5678:90ab:cdef:1234:5678:90ab:cde0"
                    },
                    "remotePortDetails": {
                        "port": 80,
                        "portName": "HTTP"
                    },
                    "localPortDetails": {
                        "port": 24198,
                        "portName": "Unknown"
                    },
                    "localNetworkInterface": "eni-abcdef00",
                    "protocol": "TCP",
                    "blocked": false
                }
            },
            "resourceRole": "ACTOR",
            "additionalInfo": {
                "sample": true,
                "value": "{\"sample\":true}",
                "type": "default"
            },
            "eventFirstSeen": "2024-11-19T18:12:05.000Z",
            "eventLastSeen": "2024-11-19T18:13:42.000Z",
            "archived": false,
            "count": 2
        },
        "severity": 8,
        "createdAt": "2024-11-19T18:12:05.011Z",
        "updatedAt": "2024-11-19T18:13:42.246Z",
        "title": "The EC2 instance i-99999999 is behaving in a manner that may indicate it is being used to perform a Denial of Service (DoS) attack using the TCP protocol.",
        "description": "The EC2 instance i-99999999 is behaving in a manner that may indicate it is being used to perform a Denial of Service (DoS) attack using the TCP protocol."
    }
}
```

:::

上記、GuardDutyのEvent例をもとに、Bedrockのドキュメントとチャット(Chat with your document)を実行するStep Functionsを作成します

:::message alert
S3のURI と SNSのトピックARN は適宜変更してください
:::

```json
{
    "Comment": "Translate GuardDuty threat detection findings into Japanese, provide remediation steps, and send them via email notifications.",
    "StartAt": "RetrieveAndGenerate",
    "States": {
        "RetrieveAndGenerate": {
            "Type": "Task",
            "Parameters": {
                "Input": {
                    "Text.$": "States.Format('あなたはAWS GuardDutyの脅威検出結果を分析するエージェントです。\nAWS GuardDutyの脅威検出結果を提供するので、対応方法の番号付きリストを使用して、脅威検出結果についての対応方法が分かるように、日本語に翻訳したうえで、原因と対応方法を簡潔に回答してください。\n\n以下がAWS GuardDutyの脅威検出結果です:\n{}', $.detail.description)"
                },
                "RetrieveAndGenerateConfiguration": {
                    "Type": "EXTERNAL_SOURCES",
                    "ExternalSourcesConfiguration": {
                        "ModelArn": "arn:aws:bedrock:us-west-2::foundation-model/anthropic.claude-3-sonnet-20240229-v1:0",
                        "Sources": [
                            {
                                "SourceType": "S3",
                                "S3Location": {
                                    "Uri": "S3のURI"
                                }
                            }
                        ]
                    }
                }
            },
            "Resource": "arn:aws:states:::aws-sdk:bedrockagentruntime:retrieveAndGenerate",
            "ResultPath": "$.result",
            "ResultSelector": {
                "translate.$": "$.Output.Text"
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
            "Resource": "arn:aws:states:::aws-sdk:sns:publish",
            "Parameters": {
                "Subject.$": "$.['detail-type']",
                "Message.$": "States.Format('日本語:\n{}\n\n\naccount: {}\ntype: {}\ntime: {}\nregion: {}\nresources: {}\nseverity: {}\n英語:\n{}', $.result.translate, $.account, $.detail.type, $.time, $.region, $.resources, $.detail.severity, $.detail.description)",
                "TopicArn": "SNSのトピックARN"
            },
            "End": true
        }
    }
}
```

`コード` を選択して、上記コードをコピペし、`SNSのトピックARN` と `S3のURI` を修正します

![](/images/20241118_aws-bedrock-guardduty/21.png)
![](/images/20241118_aws-bedrock-guardduty/22.png)

`設定` を選択して、`ステートマシン名` を入力して作成します
![](/images/20241118_aws-bedrock-guardduty/23.png)

自動で作成されるIAMロールの権限が足りないため、`AmazonBedrockFullAccess` と `AmazonS3FullAccess` のIAMポリシーを追加します
![](/images/20241118_aws-bedrock-guardduty/24.png)
![](/images/20241118_aws-bedrock-guardduty/25.png)
![](/images/20241118_aws-bedrock-guardduty/26.png)
![](/images/20241118_aws-bedrock-guardduty/27.png)
![](/images/20241118_aws-bedrock-guardduty/28.png)

#### Step Functionsのテスト

Step Functionsの動作確認を行います
`デザイン` から `RetrieveAndGenerateのフロー` を選択して、`テスト状態` を選択します
![](/images/20241118_aws-bedrock-guardduty/29.png)

`状態の入力` に、前述したGuardDutyのEvent例を入力し、`テストを開始` を選択します
成功し、日本語の結果が出力されていればOKです
![](/images/20241118_aws-bedrock-guardduty/30.png)

### EventBridge

トリガーとしてEventBridgeのルールを作成します
![](/images/20241118_aws-bedrock-guardduty/31.png)
![](/images/20241118_aws-bedrock-guardduty/32.png)

`イベントパターン` に以下を入力します

```json
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"],
  "detail": {
    "severity": [{
      "numeric": [">=", 7]
    }]
  }
}
```

![](/images/20241118_aws-bedrock-guardduty/34.png)
![](/images/20241118_aws-bedrock-guardduty/35.png)
![](/images/20241118_aws-bedrock-guardduty/37.png)

## テスト

`CloudShell` で、以下のコマンドを実行して、サンプルのGuardDutyの脅威検出結果を作成します

```bash
aws guardduty create-sample-findings --detector-id $(aws guardduty list-detectors --query 'DetectorIds' --output text) --finding-types Backdoor:EC2/DenialOfService.Tcp
```

![](/images/20241118_aws-bedrock-guardduty/38.png)
![](/images/20241118_aws-bedrock-guardduty/39.png)
※脅威検出結果がEventBridgeに流れて、メール送信されるまで最大5分程度かかります
![](/images/20241118_aws-bedrock-guardduty/40.png)

## 東京リージョンで実装する場合

以下3つが考えられると思います

- OpenSearch ServerlessやAuroraでベクトルデータベースを作成する
- 東京リージョンのEventBridgeからオレゴンリージョンのEventBridgeにEventを転送する
- 東京リージョンのEventBridge -> Lambda -> オレゴンリージョンのStep Functionsを実行 という流れにする

※EventBridgeのリージョン間転送については以下の記事が参考になります

https://dev.classmethod.jp/articles/eventbridge-cross-region-expands/

## 最後に

今後は、生成AIの画像読み取り機能も活用していきたいと感じました
先日紹介したPython Jinjaのようなテンプレートエンジンと組み合わせると、より便利になるのではないかと思いました

https://zenn.dev/metalmental/articles/20241103_jinja-cloudformation