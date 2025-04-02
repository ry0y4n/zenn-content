---
title: "自分だけの GitHub Copilot を開発してみよう"
emoji: "👨‍💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["github", "typescript", "vscode", "githubcopilot", "copilot"]
published: true
publication_name: "microsoft"
---

# はじめに

本記事では、VS Code 拡張機能を活用し、**GitHub Copilot 用のカスタム コマンドを開発する手順**を詳しく解説します。

拡張機能作成の基礎から、カスタム コマンドの実装、LLM との連携、さらには配布方法まで、実践的な内容を網羅していますので、ぜひ最後までお付き合いください。

現状、GitHub Copilot には以下のようなコマンドが用意されています（[参考](https://docs.github.com/ja/copilot/using-github-copilot/copilot-chat/github-copilot-chat-cheat-sheet#chat-participants)）。

-   **@azure**：Azure サービスに関するコンテキストと支援を提供します。
-   **@github**：GitHub 固有の Copilot スキルを利用可能にします。
-   **@terminal**：VS Code ターミナル・シェルに関するコマンドやデバッグ支援を行います。
-   **@vscode**：VS Code のコマンドや機能に関するヘルプを提供します。
-   **@workspace**：ワークスペース内のコードに関するコンテキストを補助します。

今回の記事では、これらのコマンドを参考に『**ウェブ アプリのソース コードから脆弱性を発見する**』カスタム コマンド（**@security-checker**）の実装方法を紹介します。

![alt](/images/develop-your-github-copilot/my-github-copilot-final-status.gif)

# 前提条件

-   Node.js および npm
-   VS Code
-   基本的な TypeScript の知識

# 拡張機能の初期セットアップ

以下のコマンドで、拡張機能の雛形を生成します。

```shell
npx --package yo --package generator-code -- yo code
```

対話形式の質問に答えて、以下のようにプロジェクトを作成してください。

```shell
# 今回は TypeScript / npm で拡張機能を作成します
? What type of extension do you want to create? New Extension (TypeScript)
? What's the name of your extension? my-github-copilot
? What's the identifier of your extension? my-github-copilot
? What's the description of your extension? my custom github copilot
? Initialize a git repository? Yes
? Which bundler to use? unbundled
? Which package manager to use? npm
```

プロジェクト生成後、以下のコマンドでプロジェクト フォルダに移動し、VS Code を起動します。

```shell
cd my-github-copilot
code .
```

## 拡張機能の実行

拡張機能のソース コードを開いた VS Code で `F5` を押すと、拡張機能が有効化された VS Code が起動します。

`Ctrl + Shift + P` を押してコマンド パレットを開き、**`Hello World`** コマンドを実行します。

![alt text](/images/develop-your-github-copilot/initial-command.png)

![alt text](/images/develop-your-github-copilot/initial-command-result.png)

![alt](/images/develop-your-github-copilot/develop-your-copilot_initial-status.gif)

## 現状の確認

現状のコードで確認すべきは 以下の 2 点です。

-   `package.json` にコマンドが定義されていること
-   `src/extension.ts` にコマンドの中身としての実装があること

```json
// package.json
{
    "contributes": {
        // コマンド パレットから実行できるコマンドを定義
        "commands": [
            {
                "command": "my-github-copilot.helloWorld",
                "title": "Hello World"
            }
        ]
    }
}
```

```typescript
// src/extension.ts
import * as vscode from "vscode";

// 拡張機能がアクティブになったときに呼び出される関数
export function activate(context: vscode.ExtensionContext) {
    console.log(
        'Congratulations, your extension "my-github-copilot" is now active!'
    );

    // コマンドを登録し、中身を実装
    const disposable = vscode.commands.registerCommand(
        "my-github-copilot.helloWorld",
        () => {
            // メッセージ ウィンドウを表示
            vscode.window.showInformationMessage(
                "Hello World from my-github-copilot!"
            );
        }
    );

    context.subscriptions.push(disposable);
}

export function deactivate() {}
```

# カスタム コマンドの実装

拡張機能内に、GitHub Copilot 用のカスタム コマンドを実装します。  
ここでは、基本的なコマンド登録と、チャット インターフェースでのレスポンス出力の例を示します。

冒頭でも頭出しした通り、今回は『**ウェブ アプリのソース コードから脆弱性を発見する**』カスタム コマンドを実装していきます。

## 1. コマンド登録

拡張機能の `package.json` に、以下のようなコマンド定義を追加します。

**変更前**

```json
"contributes": {
    "commands": [
        {
            "command": "my-github-copilot.helloWorld",
            "title": "Hello World"
        }
    ]
},
```

**変更後**

```json
"contributes": {
    // GitHub Copilot Chat から呼びだせるコマンドを定義
    "chatParticipants": [
        {
            "id": "security-checker", // src/extension.ts でハンドラーと紐づけるために使うユニークな ID
            "fullName": "Check Security", // コマンドの表示名
            "name": "security-checker", // @security-checker の形式で呼び出せる
            "description": "コードの脆弱性をチェックします" // @security-checker のプレースホルダー
        }
    ]
},
```

## 2. ハンドラー実装

TypeScript で、コマンドの挙動を実装します。

以下は、サンプルとなるハンドラー実装例です。

```typescript
import * as vscode from "vscode";

export function activate(context: vscode.ExtensionContext) {
    // コマンドの中身となるハンドラー関数
    const handler = async (
        request: vscode.ChatRequest, // ユーザーのプロンプトや添付ファイルなどが含まれる
        context: vscode.ChatContext, // 過去のチャット履歴などが含まれる
        stream: vscode.ChatResponseStream, // ユーザーへのレスポンスをストリーミングするためのオブジェクト
        token: vscode.CancellationToken // 操作に対し、ユーザーの意図に応じたキャンセルを行うためのトークン
    ) => {
        // stream.markdown で静的なレスポンスを返す（LLM はまだ使っていない）
        stream.markdown("Sample Response");
    };

    // コマンドを登録し、ハンドラーを紐づける
    const securityChecker = vscode.chat.createChatParticipant(
        "security-checker",
        handler
    );
}

export function deactivate() {}
```

![alt text](/images/develop-your-github-copilot/sample-response.png)

## 3. LLM の応答を取得

応答を返す方法が分かったところで、次は LLM を使って応答を取得してみましょう

```typescript
const handler = async (
    request: vscode.ChatRequest,
    context: vscode.ChatContext,
    stream: vscode.ChatResponseStream,
    token: vscode.CancellationToken
) => {
    // 専用の型を使ってメッセージ配列を初期化
    const messages: vscode.LanguageModelChatMessage[] = [];
    // ユーザーからのプロンプトをメッセージ配列に追加
    messages.push(vscode.LanguageModelChatMessage.User(request.prompt));
    // LLM にリクエストを送信
    const chatResponse = await request.model.sendRequest(messages, {}, token);

    // LLM からのレスポンスをストリーミングで受け取り、ユーザーに返す
    for await (const fragment of chatResponse.text) {
        stream.markdown(fragment);
    }

    return;
};
```

![alt text](/images/develop-your-github-copilot/llm-response.png)

## 4. システム プロンプトの追加

次は、LLM にシステム プロンプトを追加して、脆弱性を検出するためのチェックリストを提供します。

チェックリストは、[こちら](https://qiita.com/mamono210/items/0e71003c29d806c670a8) の記事の内容を使用させていただきました。

```typescript
const BASE_PROMPT = `
あなたはウェブアプリ開発におけるセキュリティの専門家です。以下の {# チェックリスト} を参考にし、セキュリティ上の問題点を指摘してください。問題点がある場合は、修正案も提示してください。

# チェックリスト

## 1. インジェクション（Injection）
### チェックポイント:

- ユーザー入力を信頼しない。
- パラメータ化クエリ（Prepared Statement）を使用する。
- 入力値を適切に検証・サニタイズする。

## 2. 認証の不備（Broken Authentication）
### チェックポイント:

- セッションIDを安全に生成・管理する。
- パスワード保存には安全なハッシュアルゴリズム（例: bcrypt）を使用する。
- 多要素認証（MFA）を導入する。

## 3. 機密情報の露出（Sensitive Data Exposure）
### チェックポイント:

- 機密情報は暗号化する（例: TLS、AES）。
- 安全な通信プロトコル（HTTPS）を強制する。
- 機密データをログに記録しない。

## 4. XML外部エンティティ（XXE）
### チェックポイント:

- 外部エンティティを無効化する。
- XMLパーサーの設定をセキュアにする。
- XMLの代わりにJSONを使用することを検討する。

## 5. アクセス制御の不備（Broken Access Control）
### チェックポイント:

- ロールベースまたは属性ベースのアクセス制御を実装する。
- デフォルトでアクセスを拒否するポリシーを設定する。
- サーバーサイドでアクセス権を検証する。

## 6. セキュリティ設定ミス（Security Misconfiguration）
### チェックポイント:

- 不要な機能やサービスを無効化する。
- アップデートとパッチ適用を定期的に行う。
- 詳細なエラーメッセージをユーザーに表示しない。

## 7. クロスサイトスクリプティング（XSS）
### チェックポイント:

- HTMLエンティティのエスケープ処理を行う。
- JavaScriptのコンテンツセキュリティポリシー（CSP）を設定する。
- ユーザー入力を適切にサニタイズする。

## 8. 不十分な安全性のあるデシリアライズ（Insecure Deserialization）
### チェックポイント:

- 信頼できないデータをデシリアライズしない。
- デシリアライズ時のデータの検証を行う。
- JSON Web Token（JWT）など安全な形式を使用する。

## 9. 使用済みコンポーネントの脆弱性（Using Components with Known Vulnerabilities）
### チェックポイント:

- 使用しているライブラリやフレームワークの脆弱性を定期的に確認する。
- 必要に応じて最新版にアップデートする。
- 不要な依存関係を削除する。

## 10. 不十分なログとモニタリング（Insufficient Logging & Monitoring）
### チェックポイント:

- セキュリティ関連のイベントを適切に記録する。
- 不正アクセスや異常をリアルタイムで検知する仕組みを導入する。
- ログデータを安全に保管する。
`;
```

上記のプロンプトを使用してメッセージを構築していきます。

```typescript
export function activate(context: vscode.ExtensionContext) {
    const handler = async (
        request: vscode.ChatRequest,
        context: vscode.ChatContext,
        stream: vscode.ChatResponseStream,
        token: vscode.CancellationToken
    ) => {
        // 上記プロンプトと共に、メッセージ配列を初期化
        const messages: vscode.LanguageModelChatMessage[] = [
            vscode.LanguageModelChatMessage.User(BASE_PROMPT),
        ];

        messages.push(vscode.LanguageModelChatMessage.User(request.prompt));

        const chatResponse = await request.model.sendRequest(
            messages,
            {},
            token
        );

        for await (const fragment of chatResponse.text) {
            stream.markdown(fragment);
        }
    };

    const securityChecker = vscode.chat.createChatParticipant(
        "security-checker",
        handler
    );
}

export function deactivate() {}
```

ユーザープロンプトにソースコードを追加して、脆弱性をチェックしてもらいましょう。

![alt](/images/develop-your-github-copilot/add-base-prompt.png)

:::details 今回使ったソースコード例

[こちら](https://www.exploit-db.com/exploits/46386) のサイトを参考にしました。

```python
from flask import Flask, request
from jinja2 import Template

app = Flask(__name__)

@app.route("/")
def index():
    username = request.values.get('username', '')
    return Template('Hello ' + username).render()

if __name__ == "__main__":
    app.run(host='127.0.0.1', port=4444)

```

:::

上記のように、きちんと脆弱性を指摘し、修正案まで提示してくれました。

## 5. 開いているファイルの自動参照

現状だと、脆弱性を見つけたいソースコードをユーザーが手動で入力する必要があります。ただ、これは GitHub Copilot で普段使われている方は分かる通り、あまり効率的ではありません。

ユーザーが開いているソースコードを自動で取得し、LLM に渡すことで、よりスムーズに脆弱性チェックを行うことができます。

実装例は以下の通りです。

```typescript
export function activate(context: vscode.ExtensionContext) {
    // アクティブなファイルを取得する関数
    const getCurrentSourceCode = () => {
        // 現在アクティブなエディタを取得
        const activeEditor = vscode.window.activeTextEditor;
        if (activeEditor) {
            const sourceCode = activeEditor.document.getText();
            const fileName = activeEditor.document.fileName;
            const fileUri = vscode.Uri.file(fileName);
            return {
                hasActiveFile: true, // アクティブなファイルがあるかどうか
                sourceCode, // ソースコード
                fileUri, // ファイルの URI
            };
        }
        return {
            hasActiveFile: false,
            sourceCode: "",
            fileUri: vscode.Uri.file(""),
        };
    };

    const handler = async (
        request: vscode.ChatRequest,
        context: vscode.ChatContext,
        stream: vscode.ChatResponseStream,
        token: vscode.CancellationToken
    ) => {
        const messages: vscode.LanguageModelChatMessage[] = [
            vscode.LanguageModelChatMessage.User(BASE_PROMPT),
        ];

        // アクティブなファイル情報を取得
        const { hasActiveFile, sourceCode, fileUri } = getCurrentSourceCode();

        let userPrompt = "";
        if (hasActiveFile) {
            // アクティブなファイルがある場合、ユーザー プロンプトにソース コードを追加
            userPrompt = `${request.prompt}\n\n# ソース コード\n\`\`\`\n${sourceCode}\`\`\``;
            // stream.reference を使うことでレスポンスに参照したファイルを表示（下記添付画像を参照）
            stream.reference(fileUri);
        } else {
            // アクティブなファイルがない場合、ユーザー プロンプトをそのまま使用
            userPrompt = request.prompt;
        }
        messages.push(vscode.LanguageModelChatMessage.User(userPrompt));

        const chatResponse = await request.model.sendRequest(
            messages,
            {},
            token
        );

        for await (const fragment of chatResponse.text) {
            stream.markdown(fragment);
        }
    };

    vscode.chat.createChatParticipant("security-checker", handler);
}
```

![alt text](/images/develop-your-github-copilot/process-references.png)

## 6. チャット履歴の活用

現状では、アクティブなファイルを取得して脆弱性チェックを行うことができましたが、過去のチャット履歴を活用することで、LLM の応答に対して追加の指示やコンテキストを提供することができます。

```typescript
// 過去のチャット履歴を取得
let previousMessages = context.history
    .map((messaage: vscode.ChatRequestTurn | vscode.ChatResponseTurn) => {
        // どちらのメッセージかを判別し、メッセージ配列の形式に変換
        if (messaage instanceof vscode.ChatRequestTurn) {
            return vscode.LanguageModelChatMessage.User(messaage.prompt);
        } else if (messaage instanceof vscode.ChatResponseTurn) {
            let fullMessage = "";
            messaage.response.forEach((fragment) => {
                if (typeof fragment.value === "string") {
                    fullMessage += fragment.value;
                } else if (fragment.value instanceof vscode.MarkdownString) {
                    fullMessage += fragment.value.value;
                }
            });
        }
        return null;
    })
    .filter((messaage) => messaage !== null);
// 過去のメッセージをメッセージ配列に追加
messages.push(...previousMessages);
```

## 7. 外部ファイルの参照

カスタム コマンドの実装シナリオとして、外部ファイルを参照したりして、いわゆる **RAG** （Retrieval-Augmented Generation）を実現することも可能です。

今回は、Azure Blob Storage に保存されたファイルを参照する例を示します。

```typescript
// Blob Storage の情報を環境変数から取得
const connectionString = process.env.AZURE_STORAGE_CONNECTION_STRING as string;
const containerName = process.env.AZURE_STORAGE_CONTAINER_NAME as string;
const blobName = process.env.AZURE_STORAGE_BLOB_NAME as string;

// Azure Blob Storage のクライアントを作成
const blobServiceClient =
    BlobServiceClient.fromConnectionString(connectionString);
const containerClient = blobServiceClient.getContainerClient(containerName);
const blobClient = containerClient.getBlobClient(blobName);

// ファイルの内容を取得
const offset = 0;
const length = undefined;
const downloadBlockBlobResponse = await blobClient.download(offset, length);
const content = await streamToText(
    downloadBlockBlobResponse.readableStreamBody as NodeJS.ReadableStream
);

// 取得したファイルの内容をシステム プロンプトに追加
const basePrompt = `あなたはウェブアプリ開発におけるセキュリティの専門家です。以下の {# チェックリスト} を参考にし、セキュリティ上の問題点を指摘してください。問題点がある場合は、修正案も提示してください。\n\n# チェックリスト\n\n${content}`;

const messages: vscode.LanguageModelChatMessage[] = [
    vscode.LanguageModelChatMessage.User(basePrompt),
];

// ストリームからテキストを取得する関数
async function streamToText(readable: NodeJS.ReadableStream): Promise<string> {
    readable.setEncoding("utf8");
    let data = "";
    for await (const chunk of readable) {
        data += chunk;
    }
    return data;
}
```

## 8. 完成したコード

ここまでの実装をまとめると、以下のようなコードになります。

```typescript
// src/extension.ts
import * as vscode from "vscode";
import fs from "fs";
import path from "path";
import { BlobServiceClient } from "@azure/storage-blob";
require("dotenv").config({ path: path.join(__dirname, "..", ".env") });

async function streamToText(readable: NodeJS.ReadableStream): Promise<string> {
    readable.setEncoding("utf8");
    let data = "";
    for await (const chunk of readable) {
        data += chunk;
    }
    return data;
}

export function activate(context: vscode.ExtensionContext) {
    const getCurrentSourceCode = () => {
        const activeEditor = vscode.window.activeTextEditor;
        if (activeEditor) {
            const sourceCode = activeEditor.document.getText();
            const fileName = activeEditor.document.fileName;
            const fileUri = vscode.Uri.file(fileName);
            return {
                hasActiveFile: true,
                sourceCode,
                fileUri,
            };
        }
        return {
            hasActiveFile: false,
            sourceCode: "",
            fileUri: vscode.Uri.file(""),
        };
    };

    const handler = async (
        request: vscode.ChatRequest,
        context: vscode.ChatContext,
        stream: vscode.ChatResponseStream,
        token: vscode.CancellationToken
    ) => {
        const connectionString = process.env
            .AZURE_STORAGE_CONNECTION_STRING as string;
        const containerName = process.env
            .AZURE_STORAGE_CONTAINER_NAME as string;
        const blobName = process.env.AZURE_STORAGE_BLOB_NAME as string;

        const blobServiceClient =
            BlobServiceClient.fromConnectionString(connectionString);
        const containerClient =
            blobServiceClient.getContainerClient(containerName);
        const blobClient = containerClient.getBlobClient(blobName);

        const offset = 0;
        const length = undefined;
        const downloadBlockBlobResponse = await blobClient.download(
            offset,
            length
        );
        const content = await streamToText(
            downloadBlockBlobResponse.readableStreamBody as NodeJS.ReadableStream
        );

        const basePrompt = `あなたはウェブアプリ開発におけるセキュリティの専門家です。以下の {# チェックリスト} を参考にし、セキュリティ上の問題点を指摘してください。問題点がある場合は、修正案も提示してください。\n\n# チェックリスト\n\n${content}`;

        const messaages: vscode.LanguageModelChatMessage[] = [
            vscode.LanguageModelChatMessage.User(basePrompt),
        ];

        let previousMessages = context.history
            .map(
                (
                    messaage: vscode.ChatRequestTurn | vscode.ChatResponseTurn
                ) => {
                    if (messaage instanceof vscode.ChatRequestTurn) {
                        return vscode.LanguageModelChatMessage.User(
                            messaage.prompt
                        );
                    } else if (messaage instanceof vscode.ChatResponseTurn) {
                        let fullMessage = "";
                        messaage.response.forEach((fragment) => {
                            if (typeof fragment.value === "string") {
                                fullMessage += fragment.value;
                            } else if (
                                fragment.value instanceof vscode.MarkdownString
                            ) {
                                fullMessage += fragment.value.value;
                            }
                        });
                    }
                    return null;
                }
            )
            .filter((messaage) => messaage !== null);

        messaages.push(...previousMessages);

        const { hasActiveFile, sourceCode, fileUri } = getCurrentSourceCode();

        let userPrompt = "";
        if (hasActiveFile) {
            userPrompt = `${request.prompt}\n\n# ソース コード\n\`\`\`\n${sourceCode}\`\`\``;
            stream.reference(fileUri);
        } else {
            userPrompt = request.prompt;
        }

        messaages.push(vscode.LanguageModelChatMessage.User(userPrompt));

        const chatResponse = await request.model.sendRequest(
            messaages,
            {},
            token
        );

        for await (const fragment of chatResponse.text) {
            stream.markdown(fragment);
        }

        return;
    };

    const securityChecker = vscode.chat.createChatParticipant(
        "security-checker",
        handler
    );
}

export function deactivate() {}
```

# 配布方法

拡張機能をパッケージ化するには、以下のコマンドを実行します。

```shell
npx vsce package
```

生成された `.vsix` ファイルを配布することで、使用者は VS Code の「**Extensions: Install from VSIX...**」機能を使ってインストールできます。

![alt](/images/develop-your-github-copilot/install-from-vsix.png)

# まとめ

今回の記事では以下のことを紹介しました。

-   **VS Code 拡張機能の初期セットアップ方法**
-   **GitHub Copilot 用カスタムコマンドの実装**
-   **ソースコードやチャット履歴、外部ファイルの参照などの各種機能の実装**
-   **拡張機能の配布方法**

今後は要件に応じてターン ハンドリングやプロンプトの調整などを行い、より高度なカスタム コマンドを実装していくことでエンタープライズ レディな拡張機能を開発することが可能です。
