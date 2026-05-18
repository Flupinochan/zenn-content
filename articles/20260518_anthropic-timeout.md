---
title: "AWS LambdaでClaude Code Agent SDK利用時のタイムアウト解決方法"
emoji: "🦪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "claude", "lambda"]
published: true
---

## 背景

Strands Agentsやめて、ClaudeのAgent SDKをLambdaで利用しようとしたらtimeoutが発生して動作しなかった

## 結論

![timeout](/images/20260518_anthropic-timeout/timeout.png)

Lambdaの環境変数にキー: `HOME`、値: `/tmp` を設定すると解決しました

Claude Platform on AWSでも同様でした

## 余談

現場でもそうですが、エラーログが出力されていない場合がとてもしんどいです...w
