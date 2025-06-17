---
title: "【API ゲートウェイ】APIM からバックエンド API への認証にマネージド ID を使用する方法"
emoji: "🔒"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure"]
published: false
---

# はじめに

今回は、API Management (APIM) で管理するバックエンド API に対して、マネージド ID 認証を使用する方法について解説します。マネージド ID を利用することで、シークレットやキーを扱うことなく、セキュアにバックエンド API へのアクセスを制御できます。

今回紹介する認証方法を使うことで、APIM をバイパスしたバックエンド API への直接アクセスを防ぎ、APIM によるセキュリティとガバナンスの配下に置くことができます。

## 【小話】MCP 文脈での APIM

昨今、**MCP (Model Context Protocol)** が注目されていますが、リモート MCP サーバーを一元管理するための **AI ゲートウェイ** として APIM を利用することができます。

MCP サーバーが流行るのと同時に、企業では **野良** の MCP サーバーをどう管理するかが課題となってくるでしょう。この問題は生成 AI ブーム初期にも起こりました。社内で野良の生成 AI アプリ・ツールが乱立し、セキュリティやガバナンスのリスクが高まるのです。このことを「**シャドー AI**」と呼びます（参考：[シャドー AI とは](https://www.ibm.com/jp-ja/think/topics/shadow-ai)）。

そんな中で台頭してきた戦略が AI ゲートウェイを設置するということです。MCP サーバーであろうが、生成 AI サーバー（例：Azure OpenAI）だろうが、その他社内の通常 API であろうが、すべてを APIM で管理することで、セキュリティやガバナンスのリスクを低減できます（APIM の概要は [こちら](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-key-concepts) から）。

![AI ゲートウェイのイメージ](/images/apim-backend-api-auth-using-managed-id/ai-gateway.gif)

> 引用元：[Azure-Samples/AI-Gateway: APIM ❤️ AI](https://github.com/Azure-Samples/AI-Gateway)

今回は APIM からバックエンド API への認証に「マネージド ID」を使用する方法を解説しますが、これは AI ゲートウェイの一部としても利用可能です。リモート MCP サーバーを Azure Functions や Azure Container Apps で構築した際に、APIM を介してのみ MCP サーバーにアクセスできるようにするための認証方法として、マネージド ID を利用することができます。

# API Management の構築

まずは APIM を作ることから始めます。今回は一番安い「Developer」レベルで構築します。ちなみに Developer レベルの API Management には SLA が含まれていないため、運用目的で使用しないでください。

> 「Standard v2」を使った閉域された APIM を構築したい方は、私が以前執筆した以下の記事をご参照ください：
>
> - [【API Management の閉域化】プライベートな API ゲートウェイを構築しよう！](https://zenn.dev/microsoft/articles/private-apim)

それでは、APIM のリソース ページから「+ 作成」を選択して APIM を作成していきます。

![APIM リソース ページから作成](/images/apim-backend-api-auth-using-managed-id/create-apim-1.png)

次に、APIM の基本情報を入力します。

- `リソース グループ`：既存のリソース グループを選択するか、新規に作成します。
- `リージョン`：「`Japan East`」等、任意のリージョンを選択します。
- `リソース名`：任意の名前を入力します。
- `組織名`：任意の組織名を入力します。
- `管理者のメールアドレス`：APIM の管理者のメールアドレスを入力します。
- `価格レベル`：「`Developer`」を選択します。

![基本タブの入力](/images/apim-backend-api-auth-using-managed-id/create-apim-2.png)

次に、「マネージド ID」タブでマネージド ID を有効化します。これにより、APIM が Azure のリソース（バックエンド API）にアクセスするための ID を持つことができます。

![システム割り当てマネージド ID の有効化](/images/apim-backend-api-auth-using-managed-id/create-apim-3.png)

最後に、APIM の設定を確認し、「作成」を選択してリソースを作成します。

![リソースの作成](/images/apim-backend-api-auth-using-managed-id/create-apim-4.png)

これで、API Management のリソースが作成されました。APIM のリソースが作成されるまで数分かかることがありますので、しばらく待ちましょう。

# バックエンド API（Azure Functions）の構築

続きまして、APIM が認証を行うバックエンド API を構築します。今回は Azure Functions を使用して、シンプルな HTTP トリガーの関数アプリを作成します。

まずは、Azure Functions のリソースを作成します。Azure Functions のリソース ページから「+ 作成」を選択し、以下の情報を入力します。

![Azure Functions リソース ページから作成](/images/apim-backend-api-auth-using-managed-id/create-function-1.png)

次に、ホスティング オプションに「フレックス従量課金」を選択します。

![ホスティング オプションの選択](/images/apim-backend-api-auth-using-managed-id/create-function-2.png)

次に、関数アプリの基本情報を入力します。

- `リソース グループ`：APIM と同じリソース グループ
- `関数アプリ名`：任意の名前を入力します。
- `リージョン`：「`Japan East`」等、APIM と同じリージョンを選択します。
- `ランタイム スタック`：「`Python`」を選択します。
- `バージョン`：「`3.12`」を選択します。
- `インスタンス サイズ`：「`2048 MB`」を選択します。

![関数アプリの基本情報の入力](/images/apim-backend-api-auth-using-managed-id/create-function-3.png)

他の設定については、デフォルトのままで問題ありません。ただ一点、「監視」タブで Application Insights を無効にすることをお勧めします。今回のコンテンツには不要なためです。

![Application Insights の無効化](/images/apim-backend-api-auth-using-managed-id/create-function-4.png)

最後に、関数アプリの設定を確認し、「作成」を選択してリソースを作成します。

![関数アプリの作成](/images/apim-backend-api-auth-using-managed-id/create-function-5.png)

関数アプリのリソースが作成されるまで数分かかることがありますので、しばらく待ちましょう。

デプロイが完了したら、以下のコマンドを実行して、HTTP トリガーを作成・デプロイします。

> ここでは Azure Functions Core Tools を使用して、ローカルで関数アプリを開発・デプロイします。事前に [Azure Functions Core Tools](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-run-local) をインストールしておいてください。

```bash
# 専用のディレクトリを作成し、仮想環境をセットアップします。
mkdir HttpExample
cd HttpExample
python -m venv .venv
.venv\scripts\activate

# テンプレートを使って関数のセットアップを行います。
func init --python
func new --name HttpExample --template "HTTP trigger" --authlevel "anonymous"

# ローカルで関数の実行を確認することもできます。
func start

# リソースを作成した Azure サブスクリプションにログインします。
az login

# 作成された関数をデプロイします。
func azure functionapp publish <APP_NAME> # <APP_NAME> は作成した関数アプリの名前に置き換えてください。
```

デプロイが成功すると以下のように、API のエンドポイントが表示されるのでアクセスしてみてください。

```bash
[2025-06-17T08:19:48.135Z] The deployment was successful!
Functions in <関数アプリ名>:
    HttpExample - [httpTrigger]
        Invoke url: https://<関数アプリ名>.azurewebsites.net/api/httpexample
```

以下のように、ブラウザで表示されるはずです。

> この API 自体は `name` というクエリ パラメータまたはリクエスト ボディを指定すると「Hello, {name}. ~~~~」と応答し、指定が無い場合は、指定する旨を表示するものです。

![API にアクセスした結果のブラウザ画面](/images/apim-backend-api-auth-using-managed-id/http-example-result.png)

# API のインポート

次に、APIM にバックエンド API をインポートします。作成した APIM のリソース ページに移動し、「API」ブレードに移動します。「Create from Azure resource」の中から「Function App」を選択します。

![関数アプリのインポート](/images/apim-backend-api-auth-using-managed-id/import-api-1.png)

「Browse」→「選択」から、先ほど作成した関数アプリを選択します。

![関数アプリの選択](/images/apim-backend-api-auth-using-managed-id/import-api-2.png)

その後、「Create」を選択することで、バックエンド API がインポートされます。

![インポート結果](/images/apim-backend-api-auth-using-managed-id/import-api-3.png)

それでは、APIM 経由でバックエンド API にアクセスしてみましょう。「`HttpExample`」という操作を選択し、表示される「Test」タブで「Send」をクリックします。

すると、以下のようにバックエンド API からの応答が表示されます。

![APIM 経由でのバックエンド API へのテスト アクセス結果](/images/apim-backend-api-auth-using-managed-id/test-api-through-apim.png)

# バックエンド API に認証を設定

# API Management での認証用ポリシーの設定

# まとめ
