---
title: "Amazon Comprehendで文章からURLを抽出する"
emoji: "📌"
type: "tech"
topics: ["aws", "comprehend", "lambda"]
published: false
---

## Amazon Comprehend の概要

文章から「感情」「単語」「言語判定」「個人情報」などが取得可能
今回は、太字の 2 つの API を使用します
※エンティティ=単語 と考えています…
| API 名 | 検出内容 | 出力される情報 |
| --- | --- | --- |
| **DetectDominantLanguageCommand** | 文章の言語 | ja, en |
| **DetectEntitiesCommand** | 単語、Type | 単語および単語のカテゴリ(PERSON, LOCATION, OTHER) |
| DetectKeyPhrasesCommand | 重要な単語 | 単語 |
| DetectPiiEntitiesCommand | 個人を特定できる情報（PII） | 単語(PHONE, EMAIL) |
| DetectSentimentCommand | 文章全体の感情 | 感情(POSITIVE) |
| DetectSyntaxCommand | 各単語の品詞 | 各単語の品詞(NOUN) |
| DetectTargetedSentimentCommand | 単語ごと感情 | 単語ごとの感情 |
| DetectToxicContentCommand | 文章の有害度 | 有害度スコア |

## コード

Lambda (Node.js) で文章から URL を抽出します
別に Comprehend である必要はないです…
Comprehend を使ってみたかっただけです…

```typescript
import { ComprehendClient, DetectEntitiesCommand, DetectDominantLanguageCommand } from "@aws-sdk/client-comprehend";

const client = new ComprehendClient({});

////////////////////////
/// 言語を検出する関数 ///
////////////////////////
const detectLanguage = async (text) => {
  const input = { Text: text };
  const command = new DetectDominantLanguageCommand(input);
  const response = await client.send(command);
  const language = response.Languages[0].LanguageCode;
  console.log("検出言語: ", language);
  return language;
};

////////////////////////
/// URLを検出する関数 ///
////////////////////////
// URLを抽出する関数 //
const extractUrls = (text) => {
  // 「http://」or「https://」から始まる文字列を取得する正規表現
  const urlRegex = /(https?:\/\/[^\s]+)/g;
  const urls = text.match(urlRegex) || [];
  return urls;
};

// 文字列を単語区切りに分割して取得 //
const detectUrls = async (text, language) => {
  const input = {
    Text: text,
    LanguageCode: language,
  };
  const command = new DetectEntitiesCommand(input);
  const response = await client.send(command);
  console.log("検出単語: ", response);
  // Entities からテキストを抽出し、その中から URL を正規表現で抽出
  const urls = response.Entities.map((entity) => entity.Text).flatMap((text) => extractUrls(text));
  console.log("検出URL: ", urls);
  return urls;
};

/////////////////
/// メイン関数 ///
/////////////////
export const handler = async (event) => {
  try {
    // 入力メッセージ
    const inputText = event.message;
    // 言語を検出
    const language = await detectLanguage(inputText);
    // URLを検出
    const urls = await detectUrls(inputText, language);
    return {
      statusCode: 200,
      body: JSON.stringify(urls),
    };
  } catch (error) {
    console.error("エラー: ", error);
    return {
      statusCode: 500,
      body: JSON.stringify({
        message: "エラーが発生しました",
        error: error.message,
      }),
    };
  }
};
```

## 実行結果

![](/images/20240817_aws-comprehend/1.png)
![](/images/20240817_aws-comprehend/2.png)

この後は、抽出した URL から Web スクレイピングするんだ ヾ(｀・ω・´)ノ
