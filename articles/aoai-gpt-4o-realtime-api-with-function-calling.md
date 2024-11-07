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

執筆時点でプレビュー中の GPT-4o Realtime API ですが、SDK が公開されております。

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

[MS Learn](https://learn.microsoft.com/ja-jp/azure/ai-services/openai/realtime-audio-quickstart?) には書かれていませんが、[SDK](https://github.com/Azure-Samples/aoai-realtime-audio-sdk/tree/main#:~:text=Works%20with%20text%20messages%2C%20function%20tool%20calling%2C%20and%20many%20other%20existing%20capabilities%20from%20other%20endpoints%20like%20/chat/completions) には Function Calling がサポートされている旨が記載されています。

> Works with text messages, function tool calling, and many other existing capabilities from other endpoints like /chat/completions

## 使い方

会話開始時に送信する JSON に `tools` プロパティを追加します。

```TypeScript
{
  "type": "session.update",
  "session": {
    "voice": "alloy",
    "instructions": "Call provided tools if appropriate for the user's input.",
    "input_audio_format": "pcm16",
    "input_audio_transcription": {
      "model": "whisper-1"
    },
    "turn_detection": {
      "threshold": 0.4,
      "silence_duration_ms": 600,
      "type": "server_vad"
    },
    // ↓ ここに tools プロパティを追加
    tools: [
      {
          type: "function",
          name: "get_my_name",
          description: "Get the name of the user",
      },
    ],
  }
}
```

会話の中で Function を使うという判断になった場合、Function に渡す引数の検討なども行った結果、サーバーは `item.type` が `function_call` の `response.output_item.done` コマンド メッセージを送信してきます（以下例）。

```JSON
// `response.output_item.done` コマンド メッセージ
{
    "type": "response.output_item.done",
    "event_id": "event_AQz0q9o4SIl68fuMPKBXN",
    "response_id": "resp_AQz0pjscf3X2HryMovLZU",
    "output_index": 0,
    "item": {
        "id": "item_AQz0pX5Q7UAMM3z8qzLHY",
        "object": "realtime.item",
        "type": "function_call",
        "status": "completed",
        "name": "get_my_name",
        "call_id": "call_cTE3ifBo5XukndV8",
        "arguments": "{}"
    }
}
```

このメッセージを拾って、Function の処理を行います。`Conversation` のアイテムとして追加した上で、`response.create` コマンド メッセージを送信することで、Function の処理が完了したことをサーバーに通知し返答の生成を促します。

```TypeScript
case "response.output_item.done":
  const { item } = message;
  if (item.type === "function_call") {
      console.log("message", message);
      console.log("function_call", item);
      if (item.name === "get_my_name") {
          realtimeStreaming.send({
              type: "conversation.item.create",
              item: {
                  type: "function_call_output",
                  call_id: item.call_id,
                  output: get_my_name(),
              },
          });
          realtimeStreaming.send({
              type: "response.create",
          });
      }
  }
  break;
```

### 結果

![alt text](/images/aoai-gpt-4o-realtime-api-with-function-calling/get_my_name_result.png)

# まとめ

今回は引数の無い関数を呼び出す形で Function Calling を試してみましたが、実際には引数を取る関数を呼び出すことが多いと思います。引数の取り方や Function の実装方法については、SDK のドキュメントを参照してください。

:::details 参考: SDK の tools 定義例

オブジェクトを引数に取る関数の定義例です。オブジェクトのプロパティを指定することや、Enum 型もサポートされています。

https://github.com/Azure-Samples/aoai-realtime-audio-sdk?tab=readme-ov-file#api-details

```JSON
"tools": [
  {
    "type": "function",
    "name": "get_weather_for_location",
    "description": "gets the weather for a location",
    "parameters": {
      "type": "object",
      "properties": {
        "location": {
          "type": "string",
          "description": "The city and state e.g. San Francisco, CA"
        },
        "unit": {
          "type": "string",
          "enum": [
            "c",
            "f"
          ]
        }
      },
      "required": [
        "location",
        "unit"
      ]
    }
  }
]
```

:::

# 参考資料

-   [Introducing the Realtime API](https://openai.com/index/introducing-the-realtime-api/)
-   [Realtime API Docs](https://platform.openai.com/docs/guides/realtime)
-   [音声とオーディオ用の GPT-4o Realtime API (プレビュー)](https://learn.microsoft.com/ja-jp/azure/ai-services/openai/realtime-audio-quickstart?pivots=programming-language-javascript)
-   [Azure OpenAI GPT-4o Audio and /realtime: Public Preview Documentation](https://github.com/Azure-Samples/aoai-realtime-audio-sdk/tree/main)
