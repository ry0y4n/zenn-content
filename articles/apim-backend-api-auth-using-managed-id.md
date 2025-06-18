---
title: "【Azure 初心者でもできる】API Management × マネージド ID で API セキュリティ強化"
emoji: "🔒"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure", "azureapimanagement", "api", "auth", "mcp"]
published: true
---

# はじめに

こんにちはモモスケです。

今回は、**API Management (APIM) で管理するバックエンド API に対して、マネージド ID 認証を使用する方法** を解説します。マネージド ID を利用することで、シークレットやキーを扱うことなく、セキュアにバックエンド API へのアクセスを制御できます。

今回紹介する認証方法を使うことで、APIM をバイパスしたバックエンド API への直接アクセスを防ぎ、APIM によるセキュリティとガバナンスの配下に置くことができます。

![今回のアーキテクチャ図](/images/apim-backend-api-auth-using-managed-id/architecture.png)

## 【小話】MCP 文脈での APIM

昨今、**MCP (Model Context Protocol)** が注目されていますが、リモート MCP サーバーを一元管理するための **AI ゲートウェイ** として APIM を利用することができます。

MCP サーバーが流行るのと同時に、企業では **野良** の MCP サーバーをどう管理するかが課題となってくるでしょう。この問題は生成 AI ブーム初期にも起こりました。社内で野良の生成 AI アプリ・ツールが乱立し、セキュリティやガバナンスのリスクが高まるのです。このことを「**シャドー AI**」と呼びます（参考：[シャドー AI とは](https://www.ibm.com/jp-ja/think/topics/shadow-ai)）。

そんな中で台頭してきた戦略が AI ゲートウェイを設置するということです。MCP サーバーであろうが、生成 AI サーバー（例：Azure OpenAI）だろうが、その他社内の通常 API であろうが、すべてを **APIM** で管理することで、セキュリティやガバナンスのリスクを低減できます（APIM の概要は [こちら](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-key-concepts) から）。

![AI ゲートウェイのイメージ](/images/apim-backend-api-auth-using-managed-id/ai-gateway.gif)

> 引用元：[Azure-Samples/AI-Gateway: APIM ❤️ AI](https://github.com/Azure-Samples/AI-Gateway)

今回は APIM からバックエンド API への認証に「**マネージド ID**」を使用する方法を解説しますが、これは AI ゲートウェイの一部としても利用可能です。リモート MCP サーバーを Azure Functions や Azure Container Apps で構築した際に、APIM を介してのみ MCP サーバーにアクセスできるようにするための認証方法として、マネージド ID を利用することができます。

# API Management の構築

まずは APIM を作ることから始めます。今回は一番安い「Developer」レベルで構築します。ちなみに Developer レベルの API Management には SLA が含まれていないため、運用目的で使用しないでください。

> 「Standard v2」を使った閉域された APIM を構築したい方は、私が以前執筆した以下の記事をご参照ください：
>
> -   [【API Management の閉域化】プライベートな API ゲートウェイを構築しよう！](https://zenn.dev/microsoft/articles/private-apim)

それでは、APIM のリソース ページから「+ 作成」を選択して APIM を作成していきます。

![APIM リソース ページから作成](/images/apim-backend-api-auth-using-managed-id/create-apim-1.png)

次に、APIM の基本情報を入力します。

-   `リソース グループ`：既存のリソース グループを選択するか、新規に作成します。
-   `リージョン`：「`Japan East`」等、任意のリージョンを選択します。
-   `リソース名`：任意の名前を入力します。
-   `組織名`：任意の組織名を入力します。
-   `管理者のメールアドレス`：APIM の管理者のメールアドレスを入力します。
-   `価格レベル`：「`Developer`」を選択します。

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

-   `リソース グループ`：APIM と同じリソース グループ
-   `関数アプリ名`：任意の名前を入力します。
-   `リージョン`：「`Japan East`」等、APIM と同じリージョンを選択します。
-   `ランタイム スタック`：「`Python`」を選択します。
-   `バージョン`：「`3.12`」を選択します。
-   `インスタンス サイズ`：「`2048 MB`」を選択します。

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

# APIM のアプリケーション (クライアント) ID の確認

次のセクションで APIM のアプリケーション ID を使用して認証を構成するので、APIM のアプリケーション ID を確認しておきます。

Entra ID のリソース ページに移動し、検索バーに APIM の名前を入力して、サービス プリンシパルを選択します。

![APIM のサービス プリンシパルの検索](/images/apim-backend-api-auth-using-managed-id/find-apim-enterprise-app.png)

APIM のサービス プリンシパルのリソース ページに移動するので、そこに表示されている「アプリケーション ID」を確認します。**後ほど使用するのでコピーして手元に控えておいてください。**

![APIM のサービス プリンシパルのリソース ページ](/images/apim-backend-api-auth-using-managed-id/apim-enterprise-app-resource-page.png)

> APIM をはじめとした Azure リソースでマネージド ID を有効化すると、**同名のサービス プリンシパルが Entra ID に自動生成されます**。これはそのリソース専用の ID で、トークンを取得 s 知恵他の Azure サービスへ安全にアクセスする役割を担います。生成・削除はリソースのライフサイクルに連動し、既定権限はゼロなので必要な RBAC を最小権限で付与して利用します。
> 詳細は [Azure リソースのマネージド ID とは](https://learn.microsoft.com/ja-jp/entra/identity/managed-identities-azure-resources/overview) をご参照ください。

# バックエンド API に認証を設定

現状だと、APIM を経由せずに直接バックエンド API にアクセスすることが可能です。これを防ぐために、APIM からのアクセスのみを許可するようにバックエンド API に認証を設定します。

今回は **Easy Auth** を使用して認証を付与します。Easy Auth は Azure Functions や App Service で簡単に認証を設定できる機能です。詳しくは [Azure App Service および Azure Functions での認証と承認](https://learn.microsoft.com/ja-jp/azure/app-service/overview-authentication-authorization) をご参照ください。

それでは、まずは作成した Azure Functions のリソース ページに移動し、「認証」ブレードから「ID プロバイダーを追加」を選択します。

![ID プロバイダーの追加](/images/apim-backend-api-auth-using-managed-id/add-id-provider.png)

ID プロバイダーの一覧から「Microsoft」を選択します。そうすると、以下のような構成画面に移動します。各構成値は以下画像を参考にしていただければ良いですが、重要なのは「**許可されたクライアント アプリケーション**」の値です。ここに先ほどコピーした APIM のアプリケーション ID を入力します。これによって、APIM からのアクセスのみが認可されるようになります。

![ID プロバイダーの構成](/images/apim-backend-api-auth-using-managed-id/configure-id-provider.png)

構成が完了したら「追加」を選択して、Azure Functions の認証を有効化します。

この状態で、バックエンド API にブラウザからアクセスしてみると、以下のような 401 エラーが表示されます。

![401 エラーのブラウザ画面](/images/apim-backend-api-auth-using-managed-id/401-error-browser.png)

最後に、この手順によって作成されたバックエンド API のエンタープライズ アプリケーションのアプリケーション ID を確認します。この値は、APIM の認証ポリシーで使用するのでコピーして手元に控えておいてください。

![バックエンド API のアプリケーション ID](/images/apim-backend-api-auth-using-managed-id/backend-api-app-id.png)

# API Management での認証用ポリシーの設定

まだ終わってません。この状態では、APIM 経由でバックエンド API にアクセスしても、認証情報をリクエストに含めていないので、ただ横流しするだけになっているので同様に 401 エラーが返されます。

![APIM 経由でのバックエンド API へのアクセス - 401 エラー](/images/apim-backend-api-auth-using-managed-id/401-error-apim.png)

そこで、APIM 自身の権限に基づいたトークンを取得し、リクエストに含める必要があります。これを実現するために、APIM で認証用のポリシーを設定します。

先ほどインポートしたバックエンド API を選択した上で、**Inbound processing** のコード アイコン（`</>`）をクリックして、ポリシーの編集画面に移動します。

![API のデザイン ページ](/images/apim-backend-api-auth-using-managed-id/api-design-page.png)

ここで、以下のポリシーを `<base />` の下に追加します。このポリシーは、APIM のマネージド ID を使用してバックエンド API へのアクセス トークンを取得し、リクエストのヘッダーに `Authorization` ヘッダーを追加するものです。

```xml
<authentication-managed-identity resource="AD_application_id"/>
```

> `AD_application_id` には、先ほどコピーしたバックエンド API のアプリケーション ID を入力してください。

![ポリシーの XML コード](/images/apim-backend-api-auth-using-managed-id/policy-xml-code.png)

これにより、APIM は割り当てられた マネージド ID で Azure AD からアクセストークンを取得し、Authorization: Bearer <token> を付与してバックエンド API に転送します。
バックエンド側の Easy Auth は次の 2 段階でトークンを検証します。

1. **認証**：署名・発行者 (iss)・有効期限 (exp)・aud などの検証に合格すること。
2. **認可**：トークン内に含まれる呼び出し元のアプリケーション ID が `allowedClientApplications` に登録済みの APIM の アプリケーション ID と一致すること。

これらを満たさないリクエストは 401 で拒否されるため、APIM をバイパスした直接アクセスは遮断できます。

> Easy Auth では基本的に「認証」のみを行い「認可」は行いません。認可を実装するには、アプリのコードでトークン検証を行う処理を追加するか、今回のように `allowedClientApplications` を使用した組み込みの認可ポリシーを使うことができます。
>
> 詳しくは [Microsoft Entra サインインを使用するように App Service アプリまたは Azure Functions アプリを構成する](https://learn.microsoft.com/ja-jp/azure/app-service/configure-authentication-provider-aad?utm_source=chatgpt.com&tabs=workforce-configuration#authorize-requests) をご参照ください。

# テスト

それでは、APIM 経由でバックエンド API にアクセスしてみましょう。先ほどの「Test」タブで「Send」をクリックします。

![APIM 経由のテスト結果 - 200 成功](/images/apim-backend-api-auth-using-managed-id/test-api-through-apim-success.png)

無事、バックエンド API からの応答が表示されました。これで、APIM 経由でバックエンド API にアクセスできるようになりました。

ターミナルからも同様に、以下の PowerShell コマンドを実行して、APIM 経由でバックエンド API にアクセスできます。

```powershell
$apiUrl = "https://<your-apim-name>.azure-api.net/<your-azure-function-name>/HttpExample"
$subscriptionKey = "<your-subscription-key>" # API はデフォルトだとサブスクリプション キーを要求する設定なのでポータルから取得する（もしくは要求しない設定にする）

$response = Invoke-WebRequest -Uri $apiUrl -Method 'POST' -Headers @{
    "Ocp-Apim-Subscription-Key" = $subscriptionKey
}

$response.Content
```

![ターミナルでの実行結果](/images/apim-backend-api-auth-using-managed-id/test-api-through-apim-terminal.png)

# まとめ

今回は APIM とマネージド ID を組み合わせて、バックエンド API をシークレット レスかつ APIM 経由に限定して保護する方法 を紹介しました。ポイントは次の 4 つだけです。

-   **シークレット不要**：マネージド ID により資格情報の保存・更新が不要になり、運用コストと漏えいリスクを削減。
-   **APIM 経由のみ許可**：Easy Auth で APIM のアプリ ID だけを許可し、直接アクセスは 401 でブロック。
-   **AI ゲートウェイにも応用**：MCP／生成 AI サーバーを含む社内 API を一元管理し、シャドー AI 対策と統制を実現。

-   **実装 3 ステップ**
    1. APIM でマネージド ID 有効化
    2. バックエンドで Easy Auth に APIM のアプリ ID を登録
    3. APIM ポリシーで `<authentication-managed-identity>` を追加し Authorization ヘッダーを注入。
