---
title: "Amazon EC2 Image Builder入門"
emoji: "💬"
type: "tech"
topics: ["aws", "ec2"]
published: true
---

## 概要

AMIの作成・削除・共有を自動化するサービス

## 図説

![](/images/20241109_aws-ec2-image-builder/flow.png)

## 用語

- イメージパイプライン
- コンポーネント
  - ビルドコンポーネント
  - テストコンポーネント
- イメージレシピ
- インフラストラクチャ
- ディストリビューション
- イメージワークフロー
  - ビルドワークフロー
  - テストワークフロー
- ライフサイクルポリシー
- セキュリティスキャン

### イメージパイプライン

- トリガー設定(EventBridge/Cron/手動)
- セキュリティスキャン(Amazon Inspector)の有効化/無効化
- 事前に定義した以下の設定を指定
  - イメージレシピ
  - インフラストラクチャ
  - ディストリビューション
  - イメージワークフロー

### コンポーネント

YAML形式で記載され、AMIを作成するために実行するコマンド等を記載する
ビルドとテストの2種類がある

- ビルドコンポーネント
  - イメージレシピのソースイメージ(AMI)から起動
  - コマンド(ShellScript、PowerShell)を記述し、インストール等を行う
- テストコンポーネント
  - 作成したAMIからEC2を起動
  - コマンド等でテストを行う
- Amazon提供のコンポーネント名一覧
  - CloudWatch Agentのインストール (amazon-cloudwatch-agent-linux)
  - Corretto(Java)のインストール (amazon-corretto-11)
  - Python3のインストール (python-3-linux)

### イメージレシピ

- ソースイメージを指定(カスタムAMIも指定可能)
- ビルドコンポーネント/テストコンポーネントを指定
- SSMエージェントの有無
- EC2ユーザデータの指定
- 作業ディレクトリ(/tmp)の指定

### インフラストラクチャ

- EC2(AMI以外)の設定
  - インスタンスタイプ
  - IAMロール
  - VPC、サブネット、セキュリティグループ
  - アラート通知先SNSトピック
  - ログ出力先S3バケット

### ディストリビューション

- AMIの作成先アカウント・リージョンの指定
- AMIを共有するアカウント・OUの指定
- 起動テンプレートの指定

### イメージワークフロー

メインの設定はコンポーネントの設定であり、イメージワークフローはデフォルトでも問題ない

- AMIを作成する処理をコンポーネントと同様にYAML形式で記述する
  - ビルドワークフロー
    - ExecuteComponentsで、ビルドコンポーネントが実行される
    - build-image(デフォルト)
      - EC2の起動
      - ビルドコンポーネントの実行
      - sysprep
      - AMIの作成
      - EC2の終了
  - テストワークフロー
    - ExecuteComponentsで、テストコンポーネントが実行される
    - test-image(デフォルト)
      - EC2の起動
      - Amazon Inspectorのスキャンを実行
      - テストコンポーネントの実行
      - EC2の終了

### ライフサイクルポリシー

- 作成したAMI/Snapshotの自動削除
  - AMIが作成されてからの経過日数
  - 作成したAMIの個数

## 補足

- 以下のCloudWatch Logs ロググループにログが出力される
  - /aws/imagebuilder/ImageName
- EC2 Image Builder実行中に、EC2 Instanceのコンソールを確認すると、AMI作成用のEC2(ビルド用とテスト用の2台)が起動していることが確認できる

## 構築例

### コンポーネントの作成

ビルドコンポーネントを作成します

![](/images/20241109_aws-ec2-image-builder/1.png)
![](/images/20241109_aws-ec2-image-builder/2.png)

以下のYAMLを記述します
※ifやfor文など様々な例を記載しています

```yaml
# https://docs.aws.amazon.com/imagebuilder/latest/userguide/toe-use-documents.html
name: tutorial
description: created by metalmental
schemaVersion: 1
# 実行時にオーバーライド可能な変数を定義
parameters:
  - MyVariableParameter:
      type: string
      default: 'Variable'
      description: Variable value
# 定数を定義
constants:
  - MyConstantParameter:
      type: string
      value: 'Constant'
# phaaseには、「build」「validate」「test」の3つがある
# 任意のnameを設定できるが、最低限以下3つのいずれかを含める必要がある
## ・build: ビルドコンポーネントで使用
## ・validate: ビルドコンポーネントで使用
## ・test: テストコンポーネントで使用
phases:
  - name: build
  # - name: validate
  # - name: test
    # stepは、作業単位(コマンド)で、入力/出力があり、後続のstepと連携可能
    # 任意のnameが設定可能
    steps:
    - name: UpdateOS
      # action名は決まっており、Linuxコマンドを実行するなら、「action: ExecuteBash」と指定する
      # windowsなら、「action: ExecutePowerShell」
      # https://docs.aws.amazon.com/imagebuilder/latest/userguide/toe-action-modules.html
      action: UpdateOS # OSのアップデートを行うaction
    - name: RunCommand
      action: ExecuteBash
      inputs:
        commands:
          - echo "Running custom command"
          - cat /etc/os-release
    - name: InputOutputTest
      action: ExecuteBash
      inputs:
        commands:
        # inputs/outputsにおけるstep間の入出力の連携
        # 1. {{}}のようなチェーン式は、inputsのみで利用可能
        # 2. 引用符「"」や「'」で囲む必要がある
        # 3. 参照方法は2つ
        # {{ phase-name.step-name.outputs.variable}}
        # {{ phase-name.step-name.outputs[index].variable}}
          # - "echo {{ build.RunCommand.outputs.OutputValue }}"
          # ExcetuteBashアクションのoutputsは、stdoutと決まっている
          - "echo outputs: {{ build.RunCommand.outputs.stdout }}"
          - "echo outputs[0]: {{ build.RunCommand.outputs[0].stdout }}" # 複数ある場合はインデックスで指定可能
    # 変数/定数の参照
    - name: OtherParameters
      action: ExecuteBash
      inputs:
        commands:
          - "echo Variable is {{ MyVariableParameter }}"
          - "echo Constant is {{ MyConstantParameter }}"
      # タイムアウト、再試行回数、エラー時の動作を指定する例
      timeoutSeconds: 120
      maxAttempts: 3
      # onFailure: Continue # エラーしたら失敗扱い(Failed)にして、次のステップを実行
      # onFailure: Ignore # エラーしても成功扱い(SuccessWithIgnoredFailure)して、次のステップを実行
      onFailure: Abort # エラー時に失敗(Failed)
    # if文を利用して、gitがインストールされていなければ、インストールする
    - name: IfSample
      action: ExecuteBash
      if:
        and:
          - binaryExists: 'git'
        # Abort, Execute, Skipのいずれかを指定 (Skipは何もしない、Executeは実行する)
        then: Skip # thenは、条件がtrueの場合に実行する
        else: Execute # elseは、条件がfalseの場合に実行する
      inputs:
        commands:
          - echo "Installing git"
          - sudo dnf install -y git
    # for文1
    - name: ForSample
      action: ExecuteBash
      loop:
        name: forSample
        for:
          start: 1
          end: 10
          updateBy: 2
      inputs:
        commands:
          - "echo ForSample"
          - "echo {{ loop.index }}: {{loop.value}}"
    # for文2
    - name: ForEachSample1
      action: ExecuteBash
      loop:
        name: forEachSample
        forEach:
          list: "element1,element2,element3"
          delimiter: ","
      inputs:
        commands:
          - "echo ForEachSample1"
          - "echo {{ loop.index }}: {{loop.value}}"
    # for文3
    - name: ForEachSample2
      action: ExecuteBash
      loop:
        name: forEachSample2
        forEach:
          - "element4"
          - "element5"
          - "element6"
      inputs:
        commands:
          - "echo ForEachSample2"
          - "echo {{ loop.index }}: {{loop.value}}"
    # YAMLファイルのため、| を使用して複数のコマンドを記述可能
    - name: EchoOSVersion
      action: ExecuteBash
      inputs:
        commands:
          - |
            FILE=/etc/os-release
            if [ -e $FILE ]; then
              . $FILE
              echo "$ID${VERSION_ID:+.${VERSION_ID}}"
            else
              echo "The file '$FILE' does not exist."
            fi

# イメージ作成中のログ出力ディレクトリ
# Linux and macOS: /var/lib/amazon/toe/
# Windows: $env:ProgramFiles\Amazon\TaskOrchestratorAndExecutor\

# 比較演算子や論理演算子もあるよ
# https://docs.aws.amazon.com/imagebuilder/latest/userguide/toe-comparison-operators.html
# https://docs.aws.amazon.com/imagebuilder/latest/userguide/toe-logical-operators.html
```

![](/images/20241109_aws-ec2-image-builder/3.png)

### イメージレシピの作成

作成したコンポーネントを元に、イメージレシピを作成します

![](/images/20241109_aws-ec2-image-builder/4.png)
![](/images/20241109_aws-ec2-image-builder/5.png)
![](/images/20241109_aws-ec2-image-builder/6.png)

### インフラストラクチャ設定の作成

IAMロール(ポリシー)は、以下3つが必要になります
※初めてイメージパイプラインを作成すると自動作成されるため、仮で1つイメージパイプラインを作成しておくのもアリです

- EC2InstanceProfileForImageBuilder
- EC2InstanceProfileForImageBuilderECRContainerBuilds
- AmazonSSMManagedInstanceCore

インスタンスタイプは、EC2の特性により、キャパシティが不足して起動できないことがあるため、複数選択することが推奨されます

- t3.micro
- t2.micro

![](/images/20241109_aws-ec2-image-builder/7.png)
![](/images/20241109_aws-ec2-image-builder/8.png)
![](/images/20241109_aws-ec2-image-builder/9.png)

### ディストリビューション設定の作成

出力AMI名には、自動的に末尾に「yyyy-mm-dd」が付与されます
共有先は、特定のアカウントやOUを指定することが可能です

![](/images/20241109_aws-ec2-image-builder/10.png)
![](/images/20241109_aws-ec2-image-builder/11.png)
![](/images/20241109_aws-ec2-image-builder/12.png)

### イメージワークフローの作成

今回はデフォルトのフローを利用するため行いません

### イメージパイプラインの作成

![](/images/20241109_aws-ec2-image-builder/13.png)

今回は、手動トリガーにします

![](/images/20241109_aws-ec2-image-builder/14.png)
![](/images/20241109_aws-ec2-image-builder/15.png)
![](/images/20241109_aws-ec2-image-builder/16.png)
![](/images/20241109_aws-ec2-image-builder/17.png)
![](/images/20241109_aws-ec2-image-builder/18.png)
![](/images/20241109_aws-ec2-image-builder/19.png)

※大阪リージョンで作成したのですが、Amazon Inspectorに対応していないようでした…

### イメージパイプラインの実行

![](/images/20241109_aws-ec2-image-builder/20.png)
![](/images/20241109_aws-ec2-image-builder/21.png)

上から2番目のステップを選択します
※「ExecuteComponents」アクションで、ビルドコンポーネントが実行されます

![](/images/20241109_aws-ec2-image-builder/22.png)
![](/images/20241109_aws-ec2-image-builder/23.png)
![](/images/20241109_aws-ec2-image-builder/24.png)


### ライフサイクルポリシーの作成


![](/images/20241109_aws-ec2-image-builder/25.png)
![](/images/20241109_aws-ec2-image-builder/26.png)
![](/images/20241109_aws-ec2-image-builder/28.png)

### AMIの確認

作成されたAMIからEC2を起動します
Amazon Linux 2023 にデフォルトでインストールされていないgitコマンドがインストールされていることが確認できました

![](/images/20241109_aws-ec2-image-builder/27.png)

## マニュアル

- [コンポーネントで使用できるアクション](https://docs.aws.amazon.com/imagebuilder/latest/userguide/toe-action-modules.html)
- [イメージワークフローで使用できるアクション](https://docs.aws.amazon.com/imagebuilder/latest/userguide/wfdoc-step-actions.html)