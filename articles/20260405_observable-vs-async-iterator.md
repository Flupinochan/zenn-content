---
title: "ObservableかAsync Iteratorか？AppSyncから学ぶリアルタイムにデータを取得する方法"
emoji: "🐑"
type: "tech"
topics: ["appsync", "graphql", "amplify", "javascript", "typescript"]
published: true
---

## はじめに

AppSyncのSubscription機能を利用するとリアルタイムなデータ更新が容易にできます

クライアント側でポーリング処理を書く必要がありません

しかし、ポートフォリオサイトなどリアルタイム性が不要な場合は、以下に気を付けるべきです

- **認証**に時間がかかる
  - AppSyncの利用には認証が必須であり、ログイン処理が不要な場合でも裏でゲスト認証させておく必要がある
- **WebSocket**の通信確立に時間がかかる
  - AppSyncの内部で利用されるWebSocketはFetchと比較して初回の通信確立に時間がかかります

GraphQLの勉強もかねて、たびたび利用していたのですが、最近は上記観点から利用を避けるようになりました

と、直近ではあまり利用していないAppSyncですが、そもそもSubscriptionとは何なのか、少し踏み込んで調べたため、メモとして残しておきます

--- 

**参考: パフォーマンス比較**

AppSyncの場合

![graphql-ms](/images/20260405_observable-vs-async-iterator/graphql-ms.png)

API Gatewayの場合

![rest-ms](/images/20260405_observable-vs-async-iterator/rest-ms.png)

## 対象者

- Subscriptionの仕組みについて知りたい方
- AppSync Subscriptionを利用する際のRepository (Interface) について知りたい方

## 解説

### Subscriptionとは

@[card](https://spec.graphql.org/September2025/#sec-Subscription)

上記GraphQLの仕様から以下のように読み取りました

- Event Stream (連続的な作成や更新等のイベント) から Response Stream (連続的なレスポンス) をし続ける操作
- 具体的な実装方法は規定されておらず、サービス側にゆだねる
  - `WebSocket` や `SSE` など通信にどのプロトコルを使用するかは明記されていない
  - `Observable` や `Async Iterator` などJavaScriptクライアント側でどのように扱うか、具体的なコードは明記されていない

まとめると、Subscriptionとは、**イベントをトリガーにして、継続的にデータを取得し続ける仕組み** の総称になるかと思います

### トランスポート層 (プロトコル) と API層

Subscriptionは、**トランスポート層** × **API層** の組み合わせで実装されます

- トランスポート層
  - `WebSocket`: サーバとクライアントの双方向通信
  - `SSE`: サーバからクライアントへの一方向通信
- API層 (JavaScriptでの実装方式)
  - `Async Iterator`: JavaScript標準
  - `Observable`: JavaScript**非標準**

例えば [graphql-js](https://github.com/graphql/graphql-js/blob/123e958de1362eef098c30e917b51981c484729e/src/execution/subscribe.ts#L53) は、JavaScript標準の `Async Iterator` で実装されています

しかし、AppSyncのクライアントライブラリである [amplify-js](https://github.com/aws-amplify/amplify-js/blob/1c911f4a198881668289509aa09eb5ff197a4187/packages/api-graphql/src/Providers/AWSWebSocketProvider/index.ts#L3) は、内部で [RxJS](https://rxjs.dev/guide/overview) を利用しており、`Observable` 形式で実装されています

そのため、AppSyncを利用するクライアント側でInterface (Repository) を定義する際は、Observableを意識して定義する必要があります

:::message
WebSocketとSSEについてはこれ以上詳しく解説しないためご了承ください
:::

### Observable vs Async Iterator

| 比較項目   | Observable                              | Async Iterator               |
| ---------- | --------------------------------------- | ---------------------------- |
| 利用方法   | 外部ライブラリ (RxJSやzen-observable等) | JavaScript標準               |
| 使い方     | `.subscribe({ next, error })`           | `for await (const x of ...)` |
| 処理モデル | Push                                    | Pull                         |

### Async IteratorがJavaScript標準となった背景

なぜ、Observable ではなく Async Iterator がJavaScript標準機能として組み込まれ、現代の主流になったのか?

主な理由は以下2点が大きいように感じられました

1. 既存の構文との親和性

Async Iterator は、JavaScript のFor文 `for...of` と似ています

- Async Iterator
  - `for await...of` でSubscriptionを定義し、非同期に連続したデータを受け取る
  - `break` や `return` でループを抜けることでSubscriptionを終了

- Observable
  - `.subscribe()` の独自メソッドで、非同期に連続したデータを受け取る
  - `.subscribe()` によって返却されたSubscriptionオブジェクトを保持しておき、そこから `.unsubscribe()` の独自メソッドを呼び出すことでSubscriptionを終了

2. Pull型の方がリスクが少ない

- Observable (Push型)
  - 挙動: サーバーからデータが届いたらコールバック関数が実行される
  - リスク: 前のデータ処理が終わる前に次のデータが届くと、処理が重なり**平行実行**されるため、ブラウザの負荷増大やメモリ不足を招く恐れがある
- Async Iterator (Pull型)
  - 挙動: `for await` の構文から分かるとおり、非同期処理を待機し、処理が完了してから**イテレート**して次のデータを処理する
  - メリット: 1つずつ確実に処理するため平行実行(処理の割り込み)のリスクがない

標準化されているのは、Pull型のAsync Iteratorですが、Observableが古い方法だったり、アンチパターンだったりするわけではありません

自分はリアルタイムという曖昧な言葉をよく使いますが、`リアルタイム = 非同期for文によるデータ取得(Async Iterator)` と考えるとしっくりきました

:::message
Observableも独自実装せず、RxJSのような外部ライブラリを適切に利用すれば上記のような並行処理によるリスクは減らせます
:::

## 【実践】AppSync Repositoryコード例

Observable方式の場合のコード例です

背景
- 音楽プレイヤーアプリを作成していた
- 音楽のメタデータ (タイトル名や音楽データ本体のS3 Prefix) をDynamoDBに保存しており、そのメタデータをAppSyncで取得していた

interfaceの定義

```typescript:interface.ts
export interface MusicMetadataRepository {
  observeMusicMetadata(): Observable<MusicMetadata[]>;
}
```

```typescript:observable.ts
export interface Observer<T> {
  next(value: T): void;
  error(error: unknown): void;
  complete(): void;
}

export interface Subscription {
  unsubscribe(): void;
}

export interface Observable<T> {
  subscribe(observer: Partial<Observer<T>>): Subscription;
}
```

implements側

```typescript:appsync.ts
export class MusicMetadataRepositoryImpl implements MusicMetadataRepository {
  constructor(private readonly client: AmplifyClient) {}

  observeMusicMetadata(): Observable<MusicMetadataDto[]> {
    return {
      subscribe: (observer: Observer<MusicMetadataDto[]>): Subscription => {
        const amplifySub = this.client.models.MusicMetadata.observeQuery({
          authMode: "identityPool",
        }).subscribe({
          next: ({ items }) => {
            const dtos = items.map((item) =>
              amplifyModelToMusicMetadataDto(item),
            );
            observer.next(dtos);
          },
          error: (e) => observer.error?.(e),
        });

        return {
          unsubscribe: () => amplifySub.unsubscribe(),
        };
      },
    };
  }
}
```


## おわりに

Subscriptionの仕組みを理解する上で、JavaScriptにおける「非同期処理 標準化」の歴史について知る必要がありました

余談になりますが、最近事件があったAxiosやjQueryについては、古い通信方式のXHRを利用しています

新しいFetchやそのラッパーライブラリであるkyの利用もより考えさせられる事件ではないかと思いました

興味がある方は、XHRとFetchの歴史も調べてみてください

最後に、生成AIにブログを添削させるとブログのタイトルが魅力的になるのはいつものことですが、最近は新しい学びも多いなと感じました

今回、添削させたことで、ObservableやAsync Iteratorを勉強するきっかけになりました

案A(自作)：AppSync SubscriptionのInterface (Repository) 定義方法
案B(生成AI)：ObservableかAsync Iteratorか？AppSyncから学ぶ非同期ストリームの設計

まあ、結局はタイトル名に「リアルタイム」という言葉を含めましたが...w

## 参考資料

amplify-jsのinterface定義について

- [SubscriptionObserver](https://aws-amplify.github.io/amplify-js/api/interfaces/aws_amplify.api._Reference_Types_.SubscriptionObserver.html)
- [Subscribable](https://aws-amplify.github.io/amplify-js/api/interfaces/aws_amplify.api._Reference_Types_.Subscribable.html)
- [Observable](https://aws-amplify.github.io/amplify-js/api/classes/aws_amplify.api._Reference_Types_.Observable.html)

tc39におけるAsync IterationとObservableの定義について

- [async iteration](https://github.com/tc39/proposal-async-iteration)
- [observable](https://github.com/tc39/proposal-observable)

Observableを学びたい方

- [RxJS](https://rxjs.dev/guide/overview)
- [ReactiveX](https://reactivex.io/intro.html)

Async Iteratorを学びたい方

- [MDN Docs](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/AsyncIterator)
