---
title: "JavaScriptコンパイル前のTypeScriptのファイル名や行数をログ出力する"
emoji: "🐡"
type: "tech"
topics: ["TypeScript", "Python", "Log", "Logger", "Logging"]
published: true
---

## 背景

私は、ログにはファイル名や行数を出力したいです
しかし、ログにファイル名や行数を出力しようとすると、コンパイル後の JavaScript のファイル名や行数が出力されてしまいました ( ;ㅿ; )

また、Python のロガー設定では、「funcName」や「lineno」とするだけで簡単に関数名や行数を出力できたのに、TypeScript のロガー設定には、そのような機能がなかったため、スタックトレースから正規表現を利用して、ファイル名や行数を取得する必要がありました

## コード例

NestJS では動きました

```typescript
import { format } from "date-fns";
import { ja } from "date-fns/locale";

// ログフォーマット
function formatLog(level: string, message: string, meta: any) {
  // JST
  const timestamp = format(new Date(), "yyyy/MM/dd HH:mm:ss.SSS OOO", {
    locale: ja,
  });
  // スタックトレース
  const stack = meta.stack || "";
  // ファイル名
  const fileName = stack.split("\n")[2]?.match(/\((.*):\d+:\d+\)/)?.[1] || "unknown";
  // 行番号
  const lineNumber = stack.split("\n")[2]?.match(/:(\d+):\d+\)/)?.[1] || "unknown";
  const lineNumberWithSuffix = lineNumber !== "unknown" ? `${lineNumber}行目` : "unknown";
  // ログメッセージ
  return `[${timestamp}] [${level}] [${fileName}] [${lineNumberWithSuffix}] ${message}`;
}

// ロガーをラップする関数
const wrapLogger = (logger: any) => {
  return {
    // DEBUGログ
    debug: (msg: string) => {
      const stack = new Error().stack || "";
      const log: any = { msg, stack };
      const formattedMessage = formatLog("DEBUG", msg, log);
      logger.debug(formattedMessage);
    },
    // INFOログ
    info: (msg: string) => {
      const stack = new Error().stack || "";
      const log: any = { msg, stack };
      const formattedMessage = formatLog("INFO", msg, log);
      logger.log(formattedMessage);
    },
    // ERRORログ
    error: (msg: string) => {
      const stack = new Error().stack || "";
      const log: any = { msg, stack };
      const formattedMessage = formatLog("ERROR", msg, log);
      logger.error(formattedMessage);
    },
  };
};

export const logger = wrapLogger(console);

// ログの出力例
logger.info("INFOログの出力例");
logger.error("ERRORログの出力例");

export default logger;
```

```bash
# 出力例
[2024/09/14 20:03:40.299 GMT+9] [INFO] [app.ts] [8行目] INFOログの出力例
[2024/09/14 20:03:40.299 GMT+9] [ERROR] [app.ts] [9行目] ERRORログの出力例
```

## 作ってみたロガーライブラリ

https://www.npmjs.com/package/metalmental-logger
https://pypi.org/project/metalmental-logger/

P.S. 自作ロガーライブラリによって、サボりがちなログ出力が捗るようになりました (╹◡╹)
