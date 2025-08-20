---
title: "【コーディング無し】既存 API サーバーを MCP サーバーに一瞬で変える方法"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure", "azureapimanagement", "api", "mcp", "llm"]
published: true
publication_name: "microsoft"
---

### 更新日: 2025/08/01

-   本機能が「AI ゲートウェイ先行アクセス」に入らなくても利用できるようになったので、
    先行アクセスの手順を削除しました。
-   本機能が v2 SKU で使用可能になったので、価格レベルの選択肢を `Basic v2` に変更しました。
-   本機能によって公開される MCP サーバーが **SSE** ではなく **Streamable HTTP** で公開されるようになったので、記事内の説明を更新しました。

# はじめに

この度、Azure API Management (APIM) で、**既存の API を MCP サーバーとして公開する機能** がプレビューでリリースされました。

https://learn.microsoft.com/ja-jp/azure/api-management/export-rest-mcp-server

この機能は先日の Microsoft Build 2025 で発表された機能です。以下がその発表の動画です。
https://www.youtube.com/watch?v=pr8ltV8XY7g

猫も杓子も MCP な時代ですが、既存の API サーバーを MCP サーバーに変えるのは中々大変です。企業人であれば、コーディング以外の大変さもあるでしょう。

![手動でコーディングして MCP 化するのと APIM の機能を使うことの比較図](/images/expose-mcp-server-in-apim/comparison.png)

今後は、APIM にインポートされている API であれば MCP の皮を被せて MCP サーバーとして公開することができるようになります。これにより以下のような恩恵を受けることができます。

-   **MCP サーバーの構築が簡単**: 既存の API を APIM にインポートするだけで MCP サーバーとして公開できます。
-   **APIM の通常恩恵**: APIM の通常の機能（認証、レート制限、ログ収集など）をそのまま利用できます。
    -   **認証**: API へのアクセス制御を簡単に設定できます（`例：社内かつ特定の社員しかアクセスさせない`）。
    -   **レート制限**: API の使用量を制限し、過負荷を防ぐことができます（`例：1 分あたりのリクエスト数上限を設ける`）。
    -   **ログ収集**: API の使用状況を詳細に把握できます（`誰がどのツールをどれくらい使用しているか`）。

APIM を使った API のキーレス認証の設定方法については、こちらの記事で詳しく解説していますので、ぜひご覧ください。

https://zenn.dev/microsoft/articles/apim-backend-api-auth-using-managed-id

また、APIM（API 群を含めて）を閉域化する方法については以下の記事で解説しています。

https://zenn.dev/microsoft/articles/private-apim

# API Management を作成

まずは、APIM リソース一覧ページに移動し、上部の「+ 作成」を選択します。

![Azure ポータルの APIM リソース一覧ページ](/images/expose-mcp-server-in-apim/create-apim-1.png)

基本タブにて以下のように入力します。

-   **リソース グループ**: 既存のリソース グループを選択するか、新規作成します。
-   **リージョン**: APIM を配置するリージョンを選択します。例: `Japan East`
-   **リソース名**: APIM の名前を入力します。例: `mcp-apim`
-   **組織名**: APIM の組織名を入力します。例: `Contoso`
-   **管理者のメール アドレス**: APIM の管理者のメールアドレスを入力します。
-   **価格レベル**: `Basic v2` を選択します。

![Azure ポータルの APIM リソース作成ページ](/images/expose-mcp-server-in-apim/create-apim-2.png)

他の設定項目はデフォルトのままにしておき、下部の「確認と作成」を選択し、「作成」から APIM の作成を開始します。

# MCP ブレードの確認

APIM の「APIs」メニュー内に「**MCP**」ブレードが追加されていることを確認します。
![Azure ポータルの APIM MCP ブレード](/images/expose-mcp-server-in-apim/mcp-blade.png)

# API をインポートし発行する

次に APIM に MCP サーバーとして公開したい API をインポートします。

今回は、[Petstore API](https://petstore3.swagger.io/) を使います。実際には、既存の社内 API や、それをホストしている Azure Functions などのリソースをインポートすることができます。

それでは、APIM の APIs ブレードに移動して「OpenAPI」を選択します。

![APIM の API インポートのホーム ページ](/images/expose-mcp-server-in-apim/import-api-1.png)

インポート画面では OpenAPI specification 欄に「**`https://petstore3.swagger.io/api/v3/openapi.json`**」を入力してください。そうすると、API の情報が自動的に取得されます。

あとは API URL suffix 欄に「**`petstore`**」を入力してください。

最後に Create を選択して API をインポートします。

![APIM の API インポート画面](/images/expose-mcp-server-in-apim/import-api-2.png)

MCP サーバーとして公開する前にこの API の動作を確認してみましょう。インポートした API 中の「**Finds Pets by status.**」という操作を選択して上部の「**Test**」タブを選択します。

「**Send**」を選択してリクエストを送信し、レスポンスが正しいことを確認します。

![APIM の API テスト画面](/images/expose-mcp-server-in-apim/test-api.png)

最後に、API の「**Settings**」タブに移動して、API のサブスクリプション キーを不要とする設定を行います。「**Subscription required**」のチェックボックスを外して、API のサブスクリプション キーを不要にします。

> もし、API のサブスクリプション キーを必要とする場合は、MCP クライアントからのリクエスト ヘッダーに `Ocp-Apim-Subscription-Key` としてキーを含める必要があります。

![API のサブスクリプション キーを不要とする設定](/images/expose-mcp-server-in-apim/api-subscription-key.png)

# MCP サーバーとして公開

それではいよいよ、インポートした API を MCP サーバーとして公開します。

「MCP Servers」ブレードに移動して、上部の「**Create new MCP Server**」→「**Expose an API as an MCP Server**」選択します。

> もう一つの選択肢に「**Expose an existing MCP Server**」があり、これは既に MCP サーバーとして公開されている API を **APIM** に取り込むためのものです。今回は新規に MCP サーバーを作成するので、前者を選択します。

先ほどインポートした API を選択し、MCP サーバーにツールとして含める操作を選択できます。ここでは「**Finds \***」系 4 つの操作を選択し、Create を押して MCP サーバーを作成します。

![APIM の MCP サーバー作成画面](/images/expose-mcp-server-in-apim/mcp-server-create.png)

公開が完了すると、MCP サーバー一覧ページに新しい MCP サーバーが表示されます。

`URL` を見ると分かる通り、MCP サーバーは `https://{your-apim-name}.azure-api.net/{MCP サーバー名}/mcp` というエンドポイントでアクセスできるようになります。そして、この MCP サーバーは **Streamable HTTP** のトランスポート形式を使用しています。

![MCP サーバー一覧ページ](/images/expose-mcp-server-in-apim/mcp-server-list.png)

# MCP Inspector から使ってみる

MCP Inspector は MCP クライアントとして MCP サーバーの動作を確認することができるツールです。MCP サーバーの動作確認や、MCP サーバーの仕様を確認するのに便利です。

```bash
npx @modelcontextprotocol/inspector
```

以下の画面を参考に、MCP サーバーの情報を入力して、ツールを実行してみましょう。

![MCP Inspector の画面](/images/expose-mcp-server-in-apim/mcp-inspector.png)

無事、ツールの実行結果が得られれば成功です。

# GitHub Copilot (VS Code) から使ってみる

次に私が普段一番使っている MCP クライアントである GitHub Copilot (VS Code) から MCP サーバーを使ってみます。

コマンド パレット（Ctrl + Shift + P）を開いて「**MCP: Add Server...**」を選択します。サーバーのタイプは `HTTP` を選択して、先ほど作成した MCP サーバーの URL を入力します。

これにより VS Code の MCP サーバー一覧に MCP サーバーが追加されます。

![VS Code の MCP サーバー一覧（settings.json）](/images/expose-mcp-server-in-apim/vscode-mcp-server-list.png)

それでは GitHub Copilot (Agent Mode) を起動して、Petstore MCP の仕様を誘導するようなプロンプト（例：`試用期間中のペットたちを教えて`）を入力してみます。

以下のように、GitHub Copilot が Petstore MCP を呼び出して結果をもとに応答してくれます。

![Petstore MCP の呼び出し結果](/images/expose-mcp-server-in-apim/call-result-petstore-mcp.png)

# 制限

-   本機能は **プレビュー** 段階です。内容は予告なく変更される可能性があるため、最新情報を随時ご確認ください。
-   API Management は MCP サーバーの「**tools**」をサポートしますが、「**resources**」および「**prompts**」には対応していません。
-   API Management の [ワークスペース](https://learn.microsoft.com/ja-jp/azure/api-management/workspaces-overview) では、MCP サーバーの機能はサポートされていません。

# まとめ

APIM の「MCP サーバーとして公開」機能を使えば、 **既存 API を "ワンクリックで AI 時代の仲間入り"** させられます。

-   **コード改修ゼロ**：APIM に取り込むだけで MCP 仕様に早変わり。
-   **運用コストも削減**：認証・レート制限・ログ統合など APIM の強力な機能をそのまま享受。
-   **クライアント連携が一気に楽に**：Copilot や自作アプリからすぐ呼び出せるので、社内外での AI 利用が加速。
-   **レガシー資産を守りつつ未来へ**：既存 API をそのまま活かし、生成 AI／MCP エコシステムへアジャスト。

**試すハードルは "数クリック"** です。プレビューの今こそ触ってみて、開発・運用フローがどこまで軽くなるか実感してみてください。
