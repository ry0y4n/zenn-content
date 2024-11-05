---
title: "【Azure OpenAI × Function Calling】"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [azure, microsoft, azureopenai, javascript]
published: false
---

# 概要

Azure OpenAI GPT-4o Realtime API で Function Calling (Tools) が使えるようなので SDK 内の JavaScript サンプル アプリに機能追加する形で実装して、その動作を確認してみます。

# Azure OpenAI の GPT-4o Realtime API おさらい

2024 年 10 月 1 日に Azure OpenAI で [GPT-4o Realtime AP](https://openai.com/index/introducing-the-realtime-api/) モデルがデプロイ可能となりました。

OpenAI のモデル自体の情報については、npaka さんの[こちらの記事](https://note.com/npaka/n/nf9cab7ea954e)が分かりやすいので、詳細は割愛させていただきますが、要するに低遅延の音声会話を実現する GPT モデルであり、それが Azure OpenAI サービスでサポートされました。

## 開発方法

執筆次点でプレビュー中の GPT-4o Realtime API ですが、SDK が公開されております。

https://github.com/Azure-Samples/aoai-realtime-audio-sdk/tree/main

以下のように WebSocket 周りの実装が抽象化されており、直観的に API サーバーとのコミュニケーションが行えるようになっています。

### Before (OpenAI API)

```JavaScript
const url = "wss://api.openai.com/v1/realtime?model=gpt-4o-realtime-preview-2024-10-01";
const ws = new WebSocket(url, {
    headers: {
        "Authorization": "Bearer " + process.env.OPENAI_API_KEY,
        "OpenAI-Beta": "realtime=v1",
    },
});
```

### After (Azure OpenAI GPT-4o Realtime API SDK)

```JavaScript
realtimeStreaming = new LowLevelRTClient(
  new URL(endpoint),
  { key: apiKey },
  { deployment: deploymentOrModel }
);
```

# Function Calling のサポート

# 参考資料

- [Introducing the Realtime API](https://openai.com/index/introducing-the-realtime-api/)
