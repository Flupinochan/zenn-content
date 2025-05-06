---
title: "HTTP通信のレジリエンス戦略"
emoji: "🐦"
type: "tech"
topics: ["csharp", "polly", "resilience"]
published: true
---

## はじめに

`耐障害性` や `可用性` の高いアプリケーションとは何か?

と問われると、インフラを主に担当してきた私にとっては、サーバの `クラスター構成` や `オートスケーリング`、`バックアップ&リストア` などのキーワードが思い浮かびます...

しかし、本記事では、フロントエンド(クライアント)側において、`耐障害性` や `可用性` の高いアプリケーションを構築する方法について紹介します!

現在、オンプレミス環境でアプリケーションを開発しているのですが、サーバのオートスケーリング機能は存在しなく... サーバの負荷によって一時的な通信断が容易に発生するからです

## レジリエンス戦略とは

AKAが謎に面白かったです...w

| 機能            | 説明                                                     | AKA                        |
| --------------- | -------------------------------------------------------- | -------------------------- |
| Retry           | エラー時にリクエストを再試行                             | たまたま調子が悪いだけかも |
| Circuit Breaker | エラー時にリクエストを一定期間停止                       | システムを休ませてあげよう |
| Timeout         | リクエストの最大処理時間を設定し、超過時にエラーをスロー | 永遠に待つわけにはいかない |
| Rate Limiter    | 一定時間内のリクエスト回数を制限                         | ちょっと落ち着こうか       |
| Fallback        | エラー時に代替処理を実行                                 | ダメでも他の方法がある     |
| Hedging         | 複数のリクエストを並行して実行し、早いレスポンスを使用   | 念のため保険をかける       |

https://www.pollydocs.org/strategies/index.html

## 実装例

C# Pollyというライブラリを使用し、回復性のあるHTTPリクエストを実装します
`Retry`、`Timeout`、`Fallback` を組み合わせ、シンプルなHTTPリクエストを行うコンソールアプリケーションを作成します
後続のHTTPリクエストがあることを考慮し、複数回リクエストします
※リトライとは別です

- `Fallback`: 通信エラーやユーザによるキャンセル操作によりHttpResponseが得られなかった場合に代替のHttpResponseに置き換える
- `OuterTimeout`: 全てのリクエスト(リトライ)における処理全体のTimeoutを設定する
- `Retry`: エラーが発生した場合にHTTPリクエストを再試行する
- `InnerTimeout`: 各リクエスト(リトライ)におけるTimeoutを設定する

![](/images/20250506_resilience-polly/1.png)

https://github.com/Flupinochan/PollySample/blob/master/PollySample/PollyConfig.cs

https://www.pollydocs.org/pipelines/index.html#sequence-diagram-for-a-pipeline-with-timeout-retry-and-timeout

1. 正常/エラー応答時1
1回目のリクエストは、リトライせずに正常で終了
2回目のリクエストは、2回目のリトライで正常で終了
3回目のリクエストは、3回リトライしたが失敗で終了
![](/images/20250506_resilience-polly/2.png)

2. 正常/エラー応答時2
1回目のリクエストは、3回リトライしたが失敗で終了 ※通信エラーではないため、後続の処理は続行
2回目のリクエストは、3回目のリトライで正常で終了
3回目のリクエストは、リトライせずに正常で終了

![](/images/20250506_resilience-polly/7.png)

3. 通信エラー時
1回目のリクエストで、3回リトライしたが通信エラーだったため、後続の処理はせずに終了
![](/images/20250506_resilience-polly/3.png)

4. 処理全体でTimeout発生時
2回目のリトライ中に処理全体のTimeoutが発生したため、3回目のリトライはせずに終了
![](/images/20250506_resilience-polly/4.png)

5. 各リクエスト(リトライ)でTimeout発生時
![](/images/20250506_resilience-polly/5.png)

6. ユーザのキャンセル操作時
サーバからのレスポンスを待たず、Fallback処理をして終了
![](/images/20250506_resilience-polly/6.png)

## おわりに

人気の生成AI、ChatGPTやClaudeなどをSaaSで利用する場合は、通信負荷などでエラー応答が高頻度で返されると思います
レジリエンス戦略を正しく構成し、ユーザエクスペリエンスを損なわないことが大切だと感じました

次回は、Progressive Web Apps を利用したオフライン対応/キャッシュ戦略を書きたいと考えています
