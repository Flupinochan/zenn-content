---
title: "AWS IAMユーザの作成とメール通知の自動化"
emoji: "😎"
type: "tech"
topics: ["aws", "stepfunctions", "iam", "secretsmanager", "ses"]
published: true
---

## 概要

AWS マネジメントコンソール(Webブラウザ)で、IAMユーザを作成し、ユーザ情報を手動でメール送信している方向けです

## 自動化内容

- Step Functions: 処理をフロー実行
- IAM: IAMユーザ作成、IAMグループ追加
- SecretsManager: パスワード生成
- SES: メール送信

## 注意点

- IAMユーザ名は、メールアドレスであること
- IAMグループは、設定済みであること
- SESは、設定済みであること
- IAM Identity Centerに移行可能であれば、移行したほうが良い

## フロー図

![](/images/20241023_aws-stepfunctions-auto-create-iamuser/1.png)

## Step Functions フロー定義

```json
{
  "Comment": "A description of my state machine",
  "StartAt": "IAMUserNamesMap",
  "States": {
    "IAMUserNamesMap": {
      "Type": "Map",
      "ItemsPath": "$.IAMUserNames",
      "Parameters": {
        "IAMUserName.$": "$$.Map.Item.Value"
      },
      "Iterator": {
        "StartAt": "IAMGroupName",
        "States": {
          "IAMGroupName": {
            "Type": "Pass",
            "Result": "NoAuthorityIAMGroup",
            "ResultPath": "$.IAMGroupName",
            "Next": "CreateUser"
          },
          "CreateUser": {
            "Type": "Task",
            "Parameters": {
              "UserName.$": "$.IAMUserName"
            },
            "Resource": "arn:aws:states:::aws-sdk:iam:createUser",
            "ResultPath": null,
            "Next": "GetRandomPassword",
            "Catch": [
              {
                "ErrorEquals": [
                  "States.ALL"
                ],
                "Next": "Pass"
              }
            ]
          },
          "GetRandomPassword": {
            "Type": "Task",
            "Parameters": {
              "PasswordLength": 10
            },
            "Resource": "arn:aws:states:::aws-sdk:secretsmanager:getRandomPassword",
            "ResultPath": "$.IAMUserPassword",
            "Next": "Choice"
          },
          "Choice": {
            "Type": "Choice",
            "Choices": [
              {
                "Or": [
                  {
                    "Variable": "$.IAMUserPassword.RandomPassword",
                    "StringMatches": "*!*"
                  },
                  {
                    "Variable": "$.IAMUserPassword.RandomPassword",
                    "StringMatches": "*@*"
                  },
                  {
                    "Variable": "$.IAMUserPassword.RandomPassword",
                    "StringMatches": "*#*"
                  },
                  {
                    "Variable": "$.IAMUserPassword.RandomPassword",
                    "StringMatches": "*$*"
                  },
                  {
                    "Variable": "$.IAMUserPassword.RandomPassword",
                    "StringMatches": "*%*"
                  },
                  {
                    "Variable": "$.IAMUserPassword.RandomPassword",
                    "StringMatches": "*^*"
                  },
                  {
                    "Variable": "$.IAMUserPassword.RandomPassword",
                    "StringMatches": "*&*"
                  },
                  {
                    "Variable": "$.IAMUserPassword.RandomPassword",
                    "StringMatches": "*(*"
                  },
                  {
                    "Variable": "$.IAMUserPassword.RandomPassword",
                    "StringMatches": "*)*"
                  },
                  {
                    "Variable": "$.IAMUserPassword.RandomPassword",
                    "StringMatches": "*_*"
                  },
                  {
                    "Variable": "$.IAMUserPassword.RandomPassword",
                    "StringMatches": "*+*"
                  },
                  {
                    "Variable": "$.IAMUserPassword.RandomPassword",
                    "StringMatches": "*-*"
                  },
                  {
                    "Variable": "$.IAMUserPassword.RandomPassword",
                    "StringMatches": "*=*"
                  },
                  {
                    "Variable": "$.IAMUserPassword.RandomPassword",
                    "StringMatches": "*[*"
                  },
                  {
                    "Variable": "$.IAMUserPassword.RandomPassword",
                    "StringMatches": "*]*"
                  },
                  {
                    "Variable": "$.IAMUserPassword.RandomPassword",
                    "StringMatches": "*{*"
                  },
                  {
                    "Variable": "$.IAMUserPassword.RandomPassword",
                    "StringMatches": "*}*"
                  },
                  {
                    "Variable": "$.IAMUserPassword.RandomPassword",
                    "StringMatches": "*|*"
                  },
                  {
                    "Variable": "$.IAMUserPassword.RandomPassword",
                    "StringMatches": "*'*"
                  }
                ],
                "Next": "CreateLoginProfile"
              }
            ],
            "Default": "GetRandomPassword"
          },
          "CreateLoginProfile": {
            "Type": "Task",
            "Parameters": {
              "Password.$": "$.IAMUserPassword.RandomPassword",
              "UserName.$": "$.IAMUserName",
              "PasswordResetRequired": true
            },
            "Resource": "arn:aws:states:::aws-sdk:iam:createLoginProfile",
            "ResultPath": null,
            "Next": "AddUserToGroup"
          },
          "AddUserToGroup": {
            "Type": "Task",
            "Parameters": {
              "UserName.$": "$.IAMUserName",
              "GroupName.$": "$.IAMGroupName"
            },
            "Resource": "arn:aws:states:::aws-sdk:iam:addUserToGroup",
            "ResultPath": null,
            "Next": "SendEmail"
          },
          "SendEmail": {
            "Type": "Task",
            "Parameters": {
              "Destination": {
                "ToAddresses.$": "States.Array($.IAMUserName)"
              },
              "Message": {
                "Subject": {
                  "Data": "AWS sign-in information"
                },
                "Body": {
                  "Text": {
                    "Data.$": "States.Format('SignInURL: https://xxxxx.signin.aws.amazon.com/console\nIAMUserName: {}\nIAMUserPassword: {}', $.IAMUserName, $.IAMUserPassword.RandomPassword)"
                  }
                }
              },
              "Source": "送信元SESメールアドレスを指定"
            },
            "Resource": "arn:aws:states:::aws-sdk:ses:sendEmail",
            "Next": "Pass"
          },
          "Pass": {
            "Type": "Pass",
            "End": true
          }
        }
      },
      "End": true
    }
  }
}
```

## Step Functions 実行時の入力パラメータ例

以下のようにIAMユーザ名(メールアドレス)を配列で指定します

```json
{
    "IAMUserNames": [
        "flupino@metalmental.net",
        "nekotto-chan@metalmental.net"
    ]
}
```

## 補足

以下のように、Secrets Managerで生成されるランダムな文字列(記号)が、IAMユーザのデフォルトのパスワードポリシー(パスワードに記号を含むこと)の要件を満たせないことがあるため、`Choice` ステートでパスワードをチェックし、満たせていない場合はパスワードを再生成しています

- Secrets Managerで生成されるランダムな文字列
  - !\"#$%&'()*+,-./:;<=>?@[\\]^_`{|}~
- IAMユーザのデフォルトのパスワードポリシー
  - !@#$%^&*()_+-=[]{}|'

https://docs.aws.amazon.com/secretsmanager/latest/apireference/API_GetRandomPassword.html
https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id_credentials_passwords_account-policy.html#default-policy-details

また、Step Functionsは正規表現が利用できないため、ワイルドカードで1つずつチェックしています…

## 実行例

![](/images/20241023_aws-stepfunctions-auto-create-iamuser/2.png)

![](/images/20241023_aws-stepfunctions-auto-create-iamuser/4.png)

![](/images/20241023_aws-stepfunctions-auto-create-iamuser/3.png)

![](/images/20241023_aws-stepfunctions-auto-create-iamuser/5.png)

![](/images/20241023_aws-stepfunctions-auto-create-iamuser/6.png)

![](/images/20241023_aws-stepfunctions-auto-create-iamuser/7.png)

## まとめ

Lambdaを使用せず、`サービスフル`なサーバレス によって自動化しました

https://iret.media/106335
https://zenn.dev/chihi_t/articles/79eccb4a8c53e6