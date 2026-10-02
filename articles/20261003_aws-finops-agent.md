---
title: "AWS FinOps Agentを使ってみた"
emoji: "💰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "finops", "awsfinopsagent", "s3"]
published: true
---

## はじめに

自分のAWSアカウントにおける先月のコストが大幅に上がっていました

いつもどおりCost ExplorerやBillingで原因を調べても良かったのですが、せっかくなので **FinOps Agent** で調べられるかどうか試してみました!

試してみたところ、想像以上に精度が良く、コスト削減に詳しくない方には **ぜひ使って欲しい** と思ったため記事にしました!

なんとなくですが、FinOps Agentのコストがほとんどかからない気がした、というのもあります

## 対象者

FinOps Agentの使用例を見てみたい方

## AWS FinOps Agentを使ってみた

### 背景

AWSにログインしBillingを確認したところ、先月(9月)のコストが上がっていることに気づきました

![1.png](/images/20261003_aws-finops-agent/1.png)

### FinOps Agentに質問

1. 9月と8月(それまで正常なコストだった月)と比較し、何のコストが上がったのか質問
   - S3 Bucketと回答された
2. S3 Bucketが原因と分かったため、どのS3 Bucketのコストが上がったのか質問
   - 2つのS3 Bucket名が回答された
     - metalmental-kubernetes-pyroscope
     - metalmental-kubernetes-tempo

![2.png](/images/20261003_aws-finops-agent/2.png)

### Cost Explorerで事実かどうか検証

1. `グループ化の条件` の `ディメンション` を `リソース` に設定
2. `フィルター` の `サービス` を `S3 (Simple Storage Service)` に設定
3. 表示される上位コストのS3 Bucket名がFinOps Agentが回答したS3 Bucket名と一致していることを確認

![3.png](/images/20261003_aws-finops-agent/3.png)

## おわりに

私はコスト削減に慣れており、Cost ExplorerやBillingでどのリソースにコストがかかっているか調べられます

そんな自分がFinOps Agentを使ってみて非常に精度が良いと感じたため、コスト削減について詳しくない方はぜひ試してみて欲しいです!

また、コスト削減に詳しいと自負している自分でも `Tier 1 (APN1-Requests-Tier1)` と `Tier 2 (APN1-Requests-Tier2)` の違いはパッと分かりません (覚えられないと思います)

その違いについてもFinOps Agentが正確に回答してくれたので素晴らしかったです

いい勉強になりました

## 補足

S3のリクエスト料金の違い

LISTを除く `読み取り`コストは、`書き込み`コストの約8% (0.00037 / 0.0047) です

| Tier                         | 対応するHTTPメソッド     | 内容                       | S3標準Bucketの場合 |
| ---------------------------- | ------------------------ | -------------------------- | ------------------ |
| Tier 1 (APN1-Requests-Tier1) | PUT / COPY / POST / LIST | データの書き込み・一覧取得 | USD 0.0047         |
| Tier 2 (APN1-Requests-Tier2) | GET / SELECT             | データの読み取り           | USD 0.00037        |

![4.png](/images/20261003_aws-finops-agent/4.png)
