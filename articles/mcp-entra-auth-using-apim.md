---
title: "MCP サーバーに Entra ID 認証を実装してセキュアな AI 連携を実現しよう"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mcp", "azure", "auth", "azureapimanagement", "entraid"]
published: true
publication_name: "microsoft"
---

## はじめに

この記事では、Azure API Management (APIM) を使用して MCP (Model Context Protocol) サーバーの認証・認可を実装する方法について解説します。

Microsoft が提供する公式サンプル コード [remote-mcp-apim-functions-python](https://github.com/Azure-Samples/remote-mcp-apim-functions-python) を基に、MCP サーバーにおける認証・認可の実装パターンを学びます。このサンプル コードには、**Entra ID 認証が実装された MCP サーバー アーキテクチャ** が含まれています。

https://github.com/Azure-Samples/remote-mcp-apim-functions-python

サンプル コードを実際に動かしながら、**MCP サーバーにおける認証・認可の実装パターン** を理解し、本番環境で使用できるセキュアな MCP サーバーの構築方法を習得することが本記事の目的です。

Azure の各サービスがどのように連携して動作するかを、具体的なコード例とともに詳しく見ていきましょう。

![APIM を使った MCP サーバーの認証・認可全体像](/images/mcp-entra-auth-using-apim/mcp-client-authorization.gif)

# 前提知識

-   MCP についての基本的な理解
-   Azure API Management (APIM) の基本的な理解
-   認証・認可の基本的な理解

## Azure API Management とは

Azure API Management (APIM) は、既存のバックエンド サービスに対して API ゲートウェイとしての機能を提供する Azure のマネージド サービスです。主な機能として以下があります：

-   **API ゲートウェイ**: クライアントとバックエンド サービス間のプロキシとして動作
-   **認証・認可**: OAuth、JWT、API キーなどによる認証機能
-   **ポリシー管理**: リクエスト・レスポンスの変換、ルーティング、レート制限など
-   **監視・分析**: API 使用状況の監視とログ収集

https://learn.microsoft.com/ja-jp/azure/api-management/api-management-key-concepts

APIM では、「ポリシー」と呼ばれる XML 形式の設定を使用して、API のリクエスト処理をカスタマイズできます。ポリシーは以下の 4 つのセクションで構成されます：

```xml
<policies>
    <inbound>
        <!-- クライアントからのリクエストを受信時に実行 -->
    </inbound>
    <backend>
        <!-- バックエンドサービスへのリクエスト送信時に実行 -->
    </backend>
    <outbound>
        <!-- バックエンドからのレスポンス受信時に実行 -->
    </outbound>
    <on-error>
        <!-- エラー発生時に実行 -->
    </on-error>
</policies>
```

APIM を使用することで、**MCP サーバー側に認証・認可を実装することなく、クライアントとサーバーの間に認証・認可のゲートウェイを設けることができます**。これにより、MCP サーバーは純粋な MCP プロトコルの実装に集中でき、セキュリティやトークン管理の複雑さを大幅に軽減できます。

# サンプル コードのセットアップ

まずはサンプル コードを使って Azure リソースの展開や MCP サーバーのデプロイを行います。

この IaC により、APIM と Azure Functions を使用した MCP サーバーの認証・認可の実装が自動的に行われます。

```bash
git clone https://github.com/Azure-Samples/remote-mcp-apim-functions-python
cd remote-mcp-apim-functions-python

azd auth login
azd up
```

次に、MCP クライアントを用意します。ここでは、MCP Inspector を使用します。

```bash
npx @modelcontextprotocol/inspector@0.10.2
```

> 最新バージョンの MCP Inspector だと上手く動作しなかったので動作が確認できたバージョンを指定しています。

トランスポート プロトコルは「SSE」を選択して、URL には `https://<your-apim-name>.azure-api.net/mcp/sse` を指定します。

![MCP Inspector の動作確認](/images/mcp-entra-auth-using-apim/mcp-auth-apim.gif)

上記動画の通り、MCP サーバーに接続しようとすると、同意画面が表示されその後 Entra ID の認証画面にリダイレクトされます（動画だとログイン済みなのでスキップされている）。

```mermaid
flowchart LR
    A[MCP クライアント] --> B[同意画面]
    B --> C[Entra ID ログイン画面]
    C --> D[MCP クライアント<br>（接続完了）]
```

# APIM にインポートされる API と操作の一覧

さて、動作確認ができたところで、APIM にインポートされる API とその操作を確認していきましょう。

今回のサンプル コードでは以下の 2 つの API が定義されています。

-   **OAuth API (`/oauth/*`)**
-   **MCP API (`/mcp/*`)**

## OAuth API (/oauth/\*)

| 操作名                | HTTP メソッド | パス                                    | 説明                               |
| --------------------- | ------------- | --------------------------------------- | ---------------------------------- |
| authorize             | GET           | /authorize                              | OAuth 認証を開始する               |
| token                 | POST          | /token                                  | 認証コードをアクセストークンと交換 |
| oauth-callback        | GET           | /oauth-callback                         | Entra ID からのコールバック処理    |
| register              | POST          | /register                               | クライアント登録                   |
| register-options      | OPTIONS       | /register                               | CORS 対応                          |
| oauthmetadata-get     | GET           | /.well-known/oauth-authorization-server | OAuth 設定情報を公開               |
| oauthmetadata-options | OPTIONS       | /.well-known/oauth-authorization-server | CORS 対応                          |
| consent-get           | GET           | /consent                                | ユーザー同意画面表示               |
| consent-post          | POST          | /consent                                | ユーザー同意処理                   |

## MCP API (/mcp/\*)

| 操作名      | HTTP メソッド | パス     | 説明                                    |
| ----------- | ------------- | -------- | --------------------------------------- |
| mcp-sse     | GET           | /sse     | Server-Sent Events でのリアルタイム通信 |
| mcp-message | POST          | /message | MCP プロトコルメッセージの送受信        |

# MCP サーバーの認証・認可の流れ

上記の API たちを使用して、MCP サーバーの認証・認可がどのように行われるかを、ステップバイステップで解説していきます。

## ステップ ①：クライアント登録（`POST /register`）

`POST /register` は、MCP クライアント（例：Claude）が API にアクセスする前に、自分自身をシステムに登録するためのエンドポイントです。これは OAuth 2.0 の動的クライアント登録仕様に基づいています。

MCP クライアントは、まずこのエンドポイントにリクエストを送信し、システムからクライアント ID を取得します。このクライアント ID は、後続の認証フローで使用されます。

以下のポリシー ファイルは、`/register` エンドポイントの実装を示しています。この中身を紐解きながら、どのようにクライアント登録が行われるかを見ていきます。

https://github.com/Azure-Samples/remote-mcp-apim-functions-python/blob/main/infra/app/apim-oauth/register.policy.xml

### 1: クライアント登録データの抽出

```xml
<set-variable name="requestBody" value="@(context.Request.Body.As<JObject>(preserveContent: true))" />
```

**何をしているか**

-   MCP クライアントから送信された JSON リクエスト ボディを抽出
-   `preserveContent: true` により、後続の処理でも同じボディを参照可能（[参考](https://learn.microsoft.com/ja-jp/azure/api-management/set-body-policy)）

**MCP クライアントが送信するデータ例（MCP Inspector）**

```json
{
    "client_name": "MCP Inspector",
    "client_uri": "https://github.com/modelcontextprotocol/inspector",
    "redirect_uris": ["http://127.0.0.1:6274/oauth/callback"]
}
```

### 2: 登録情報のキャッシュ保存

#### 2-1: 全体の登録情報を保存

```xml
<cache-store-value duration="3600"
    key="DynamicClientRegistration"
    value="@(context.Variables.GetValueOrDefault<JObject>("requestBody").ToString())" />
```

**何をしているか**

-   登録リクエスト全体を APIM のキャッシュに 1 時間（3600 秒）保存
-   キー名は固定で `DynamicClientRegistration`

#### 2-2: リダイレクト URI を個別に保存

```xml
<cache-store-value duration="3600"
    key="ClientRedirectUri"
    value="@(context.Variables.GetValueOrDefault<JObject>("requestBody")["redirect_uris"][0].ToString())" />
```

**何をしているか**

-   `redirect_uris` 配列の最初の要素を取得
-   後の OAuth 2.0 フローで、リダイレクト先を検証するために使用

#### 2-3: ユニークなクライアント ID の生成

```xml
<set-variable name="uniqueClientId" value="@(Guid.NewGuid().ToString())" />
```

**何をしているか**

-   新しい GUID を生成してクライアント ID として使用
-   例：`a1b2c3d4-e5f6-7890-abcd-ef1234567890`

#### 2-4: クライアント情報の詳細保存

```xml
<cache-store-value duration="3600"
    key="@($"ClientInfo-{context.Variables.GetValueOrDefault<string>("uniqueClientId")}")"
    value="@{
        var requestBody = context.Variables.GetValueOrDefault<JObject>("requestBody");
        var clientInfo = new JObject();
        clientInfo["client_name"] = requestBody["client_name"]?.ToString() ?? "Unknown Application";
        clientInfo["client_uri"] = requestBody["client_uri"]?.ToString() ?? "";
        clientInfo["redirect_uris"] = requestBody["redirect_uris"];
        return clientInfo.ToString();
    }" />
```

**何をしているか**

-   キー名：`ClientInfo-{生成されたクライアント ID}`
-   後の認証フローで、このクライアント ID を使用してクライアント情報を検索
-   `??` 演算子で null 値の場合のデフォルト値を設定

**保存される情報の例**

```json
{
    "client_name": "MCP Inspector",
    "client_uri": "https://github.com/modelcontextprotocol/inspector",
    "redirect_uris": ["http://127.0.0.1:6274/oauth/callback"]
}
```

### 3: レスポンスの準備

```xml
<set-header name="Content-Type" exists-action="override">
    <value>application/json</value>
</set-header>
```

**何をしているか**

-   HTTP レスポンス ヘッダーでコンテンツ タイプを JSON に設定

### 4: クライアント認証情報の返却

#### 4-1: タイムスタンプの生成

```xml
// Current time in seconds since epoch (Unix timestamp)
long currentTimeSeconds = DateTimeOffset.UtcNow.ToUnixTimeSeconds();

// Client ID issued at current time
long clientIdIssuedAt = currentTimeSeconds;

// Client secret expires in 1 year (31536000 seconds = 365 days)
long clientSecretExpiresAt = currentTimeSeconds + 31536000;
```

**何をしているか**

-   現在時刻を Unix タイムスタンプ（1970 年 1 月 1 日からの秒数）で取得
-   クライアント ID の発行時刻を記録
-   クライアント シークレットの有効期限を 1 年後に設定

#### 4-2: レスポンス JSON の構築

```xml
return new JObject
{
    ["client_id"] = uniqueClientId,
    ["client_id_issued_at"] = clientIdIssuedAt,
    ["client_secret_expires_at"] = clientSecretExpiresAt,
    ["redirect_uris"] = requestBody["redirect_uris"]?.ToObject<JArray>(),
    ["client_name"] = requestBody["client_name"]?.ToString() ?? "Unknown Application",
    ["client_uri"] = requestBody["client_uri"]?.ToString() ?? ""
}.ToString();
```

**何をしているか**

-   クライアント ID、発行時刻、有効期限、リダイレクト URI、クライアント名、クライアント URI を含む JSON オブジェクトを作成

#### 4-3: レスポンスの返却

```xml
<return-response>
    <set-status code="200" reason="OK" />
    <set-header name="access-control-allow-origin" exists-action="override">
        <value>*</value>
    </set-header>
    <set-body template="none">
        <!-- 4-1 & 4-2 の内容 -->
    </set-body>
</return-response>
```

**何をしているか**

-   HTTP ステータス 200 で成功レスポンスを返却
-   CORS ヘッダーを設定して、どのオリジンからのリクエストも許可
-   レスポンス ボディにクライアント情報を JSON 形式で設定

**MCP クライアントに返却される情報の例：**

```json
{
    "client_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "client_id_issued_at": 170000000,
    "client_secret_expires_at": 1731536000,
    "client_name": "MCP Inspector",
    "client_uri": "https://github.com/modelcontextprotocol/inspector",
    "redirect_uris": ["http://127.0.0.1:6274/oauth/callback"]
}
```

### ステップ ① まとめ

-   ✅ **クライアント登録**: OAuth 2.0 の動的クライアント登録仕様に準拠した登録処理
-   ✅ **GUID ベース ID 生成**: セキュアなユニークなクライアント ID を自動生成
-   ✅ **リダイレクト URI 管理**: 後の認証フローで使用するコールバック URI を保存
-   ✅ **キャッシュベース永続化**: 複数のキー形式でクライアント情報を保存（`ClientInfo-{id}`、`DynamicClientRegistration` 等）

このクライアント登録が完了すると、MCP クライアントは受け取った `client_id` を使って次のステップである認証フロー（`/authorize`）を開始できるようになります。

## ステップ ② 認証開始（`GET /authorize`）

`GET /authorize` は、MCP クライアントが OAuth 2.0 + PKCE 認証フローを開始するためのエンドポイントです。このエンドポイントでは、ユーザーの同意確認、PKCE パラメータの生成、そして Entra ID へのリダイレクトを行います。

https://github.com/Azure-Samples/remote-mcp-apim-functions-python/blob/main/infra/app/apim-oauth/authorize.policy.xml

### 1: MCP クライアントからの PKCE パラメータ抽出

```xml
<!-- STEP 1: Retrieve PKCE parameters from MCP Client request -->
<set-variable name="mcpClientCodeChallenge" value="@((string)context.Request.Url.Query.GetValueOrDefault("code_challenge", ""))" />
<set-variable name="mcpClientCodeChallengeMethod" value="@((string)context.Request.Url.Query.GetValueOrDefault("code_challenge_method", ""))" />
```

**何をしているか**

-   MCP クライアントが送信した PKCE パラメータを抽出
-   `code_challenge`: MCP クライアントが生成した `code_verifier` の SHA256 ハッシュ値
-   `code_challenge_method`: 通常は "S256"

**MCP クライアントからの典型的なリクエスト例：**

```
GET /authorize?response_type=code&client_id=abc123&code_challenge=xyz789&code_challenge_method=S256&redirect_uri=http://localhost:8080/callback&state=random123
```

### 2: クライアント情報とリダイレクト先の抽出

```xml
<!-- STEP 2: Extract client ID and check if consent is required -->
<set-variable name="clientId" value="@((string)context.Request.Url.Query.GetValueOrDefault("client_id", ""))" />
<set-variable name="redirect_uri" value="@((string)context.Request.Url.Query.GetValueOrDefault("redirect_uri", ""))" />
<set-variable name="consentStatus" value="@((string)context.Request.Url.Query.GetValueOrDefault("consent", ""))" />
<set-variable name="currentState" value="@((string)context.Request.Url.Query.GetValueOrDefault("state", ""))" />
```

**何をしているか：**

-   `client_id`: 前ステップの `/register` で発行されたクライアント ID
-   `redirect_uri`: 認証完了後にリダイレクトする MCP クライアントの URI
-   `consent`: 同意状況（"granted"など）
-   `state`: CSRF 攻撃防止用のランダム値

### 3: 同意状況の確認

#### 3-1: 同意クッキーの確認

```xml
<!-- Check for consent cookie -->
<set-variable name="consentCookie" value="@{
    string cookieName = $"MCP_CLIENT_CONSENT_{context.Variables.GetValueOrDefault<string>("clientId")}";
    return context.Request.Headers.GetValueOrDefault("Cookie", "")
        .Split(';')
        .Select(c => c.Trim().Split('='))
        .Where(c => c.Length == 2 && c[0] == cookieName)
        .Select(c => c[1])
        .FirstOrDefault() ?? "";
}" />
```

**何をしているか：**

-   `MCP_CLIENT_CONSENT_{clientId}` という名前のクッキーを検索
-   以前にユーザーが同意していた場合、クッキーに "granted" が保存されている

#### 3-2: 同意確認の判定

```xml
<choose>
    <when condition="@(
        context.Variables.GetValueOrDefault<string>("consentStatus") == "granted" ||
        context.Variables.GetValueOrDefault<string>("consentCookie") == "granted"
    )">
        <!-- Continue with normal flow - client is authorized -->
    </when>
    <otherwise>
        <!-- Redirect to consent page -->
        <return-response>
            <set-status code="302" reason="Found" />
            <set-header name="Location" exists-action="override">
                <value>@{
                    string basePath = context.Request.OriginalUrl.Scheme + "://" + context.Request.OriginalUrl.Host + (context.Request.OriginalUrl.Port == 80 || context.Request.OriginalUrl.Port == 443 ? "" : ":" + context.Request.OriginalUrl.Port);
                    return $"{basePath}/consent?client_id={context.Variables.GetValueOrDefault<string>("clientId")}&redirect_uri={context.Variables.GetValueOrDefault<string>("redirect_uri")}&state={context.Variables.GetValueOrDefault<string>("currentState")}";
                }</value>
            </set-header>
        </return-response>
    </otherwise>
</choose>
```

**何をしているか：**

-   URL パラメータまたはクッキーで同意確認済みの場合：認証フローを継続
-   未同意の場合：`/consent` ページにリダイレクト
    -   例：`https://your-apim.azure-api.net/consent?client_id=abc123&redirect_uri=http://localhost:8080/callback&state=random123`
    -   [ステップ 4: ユーザー同意処理 (GET/POST /consent)](#ステップ--ユーザー同意処理getpost-consent) からの処理に進む

### 4: Entra ID 用 PKCE パラメータの生成

```xml
<!-- STEP 3: Generate PKCE parameters for Entra ID authentication -->
<!-- Generate a random code verifier for Entra ID -->
<set-variable name="codeVerifier" value="@((string)Guid.NewGuid().ToString().Replace("-", ""))" />
<!-- Set the code challenge method for Entra ID -->
<set-variable name="codeChallengeMethod" value="S256" />
<!-- Generate a code challenge using SHA-256 for Entra ID -->
<set-variable name="codeChallenge" value="@{
    using (var sha256 = System.Security.Cryptography.SHA256.Create())
    {
        var bytes = System.Text.Encoding.UTF8.GetBytes((string)context.Variables.GetValueOrDefault("codeVerifier", ""));
        var hash = sha256.ComputeHash(bytes);
        return System.Convert.ToBase64String(hash).TrimEnd('=').Replace('+', '-').Replace('/', '_');
    }
}" />
```

**何をしているか：**

-   MCP クライアント用と Entra ID 用で別々の PKCE パラメータを生成
-   Entra ID 用の新しい `code_verifier` を生成（GUID ベース）
-   SHA256 ハッシュで `code_challenge` を生成し、Base64URL 形式に変換

### 5: Entra ID 認証 URL の構築

```xml
<!-- STEP 4: Construct the Entra ID authorization URL -->
<set-variable name="baseAuthUrl" value="https://login.microsoftonline.com/{{EntraIDTenantId}}/oauth2/v2.0/authorize?response_type=code" />
<set-variable name="clientIdParam" value="@("&client_id={{EntraIDClientId}}")" />
<set-variable name="codeChallengeParam" value="@(string.Concat("&code_challenge=", context.Variables.GetValueOrDefault("codeChallenge", "")))" />
<set-variable name="codeChallengeMethodParam" value="@(string.Concat("&code_challenge_method=", context.Variables.GetValueOrDefault("codeChallengeMethod", "")))" />
<set-variable name="redirectUriParam" value="@(string.Concat("&redirect_uri=", "{{OAuthCallbackUri}}" ))" />
<set-variable name="scopeParam" value="@(string.Concat("&scope={{OAuthScopes}}"))" />
<set-variable name="entraState" value="@((string)Guid.NewGuid().ToString())" />
<set-variable name="stateParam" value="@(string.Concat("&state=", context.Variables.GetValueOrDefault("entraState", "")))" />
```

**構築される典型的な URL 例：**

```
https://login.microsoftonline.com/your-tenant-id/oauth2/v2.0/authorize?
response_type=code&
client_id=entra-app-client-id&
code_challenge=xyz789&
code_challenge_method=S256&
redirect_uri=https://your-apim.azure-api.net/oauth-callback&
scope=openid%20profile%20email&
state=entra-state-guid
```

### 6: 認証データのキャッシュ保存

#### 6-1: `code_verifier` の保存

```xml
<!-- Store code verifier for token exchange -->
<cache-store-value duration="3600"
    key="@("CodeVerifier-"+context.Variables.GetValueOrDefault("entraState", ""))"
    value="@(context.Variables.GetValueOrDefault("codeVerifier", ""))" />
```

#### 6-2: 確認コードの生成と保存

```xml
<!-- Generate a confirmation code to return to the MCP client -->
<set-variable name="mcpConfirmConsentCode" value="@((string)Guid.NewGuid().ToString())" />

<!-- Map state to MCP confirmation code for callback -->
<cache-store-value duration="3600"
    key="@((string)context.Variables.GetValueOrDefault("entraState"))"
    value="@(context.Variables.GetValueOrDefault("mcpConfirmConsentCode", ""))" />
```

#### 6-3: MCP クライアント データの保存

```xml
<!-- Store MCP client PKCE data, client state and client callback redirect uri for verification during token exchange -->
<cache-store-value duration="3600"
    key="@($"McpClientAuthData-{context.Variables.GetValueOrDefault("mcpConfirmConsentCode")}")"
    value="@{
        return new JObject{
            ["mcpClientCodeChallenge"] = (string)context.Variables.GetValueOrDefault("mcpClientCodeChallenge", ""),
            ["mcpClientCodeChallengeMethod"] = (string)context.Variables.GetValueOrDefault("mcpClientCodeChallengeMethod", ""),
            ["mcpClientState"] = (string)context.Variables.GetValueOrDefault("mcpState", ""),
            ["mcpClientScope"] = (string)context.Variables.GetValueOrDefault("mcpScope", ""),
            ["mcpCallbackRedirectUri"] = Uri.UnescapeDataString(context.Variables.GetValueOrDefault("redirect_uri", ""))
        }.ToString();
    }" />
```

**キャッシュに保存される情報：**

-   `CodeVerifier-{entraState}` → Entra ID 用の `code_verifier`
-   `{entraState}` → MCP 確認コード
-   `McpClientAuthData-{mcpConfirmConsentCode}` → MCP クライアントの認証データ

### 7: Entra ID へのリダイレクト

```xml
<return-response>
    <set-status code="302" reason="Found" />
    <set-header name="Location" exists-action="override">
        <value>@(context.Variables.GetValueOrDefault("authUrl", ""))</value>
    </set-header>
</return-response>
```

**何をしているか：**

-   HTTP ステータス 302 で Entra ID の認証ページにリダイレクト
-   ユーザーは Microsoft Entra ID の認証画面に遷移

### ステップ ② まとめ

-   ✅ **同意状況の確認**: クッキーまたは URL パラメータで既存の同意状況をチェック
-   ✅ **PKCE パラメータ分離**: MCP クライアント用と Entra ID 用で独立した PKCE 情報を生成
-   ✅ **状態管理の初期化**: 複数の state 値とセッション情報をキャッシュに保存
-   ✅ **条件分岐処理**: 同意済み → Entra ID へ、未同意 → 同意画面へのルーティング

次のステップは 2 つのパスに分岐します：

-   ① ユーザーが同意済みの場合は Entra ID の認証画面にリダイレクト
-   ② 未同意の場合は `/consent` エンドポイントにリダイレクトし、ユーザー同意を得て `/authorize` に戻ってきて ① の処理を行う

```mermaid
flowchart LR
    A["/authorize"] --> B{ユーザーは同意済み？}
    B -- はい --> C[Entra ID<br>認証画面]
    B -- いいえ --> D["/consent"]
    D --> A
```

ということで、次のステップはユーザー同意を得るための `/consent` エンドポイントの実装を見ていきます。

## ステップ ③ ユーザー同意処理（`GET/POST /consent`）

`/consent` エンドポイントは、MCP クライアントが API にアクセスする際のユーザー同意を管理します。GET リクエストで同意画面を表示し、POST リクエストで同意の結果を処理します。

```mermaid
flowchart LR
    B["/consent<br>エンドポイント (GET)/"]
    B --> C[同意画面HTMLを返却]
    C --> D[ユーザーが同意ボタン押下]
    D --> E["/consent<br>エンドポイント (POST)/"]
    E --> F[同意結果を処理]
```

https://github.com/Azure-Samples/remote-mcp-apim-functions-python/blob/main/infra/app/apim-oauth/consent.policy.xml

### 1: リクエスト パラメータの抽出

```xml
<!-- Extract parameters from request -->
<set-variable name="client_id" value="@((string)context.Request.Url.Query.GetValueOrDefault("client_id", ""))" />
<set-variable name="redirect_uri" value="@((string)context.Request.Url.Query.GetValueOrDefault("redirect_uri", ""))" />
<set-variable name="state" value="@((string)context.Request.Url.Query.GetValueOrDefault("state", ""))" />
```

**受信するリクエスト例：**

```
GET /consent?client_id=abc123&redirect_uri=http://localhost:8080/callback&state=random123
```

### 2: リダイレクト URI の正規化

```xml
<!-- Explicitly normalize the redirect_uri by decoding it once -->
<set-variable name="normalized_redirect_uri" value="@{
    string redirectUri = context.Variables.GetValueOrDefault<string>("redirect_uri", "");
    // Single decode is enough - we want to work with the normal decoded version everywhere
    return System.Net.WebUtility.UrlDecode(redirectUri);
}" />
```

**何をしているか：**

-   URL エンコードされたリダイレクト URI をデコード

### 3: クライアント登録確認

```xml
<!-- Look up client information from cache -->
<cache-lookup-value key="@($"ClientInfo-{context.Variables.GetValueOrDefault<string>("client_id")}")" variable-name="clientInfoJson" />

<!-- Check if client exists in cache -->
<set-variable name="is_client_registered" value="@{
    try {
        string clientId = context.Variables.GetValueOrDefault<string>("client_id", "");
        string redirectUri = context.Variables.GetValueOrDefault<string>("normalized_redirect_uri", "");

        if (string.IsNullOrEmpty(clientId)) {
            return false;
        }

        // Get the client info from the variable set by cache-lookup-value
        string clientInfoJson = context.Variables.GetValueOrDefault<string>("clientInfoJson");
        if (string.IsNullOrEmpty(clientInfoJson)) {
            context.Trace($"Client info not found in cache for client_id: {clientId}");
            return false;
        }

        // Parse client info
        JObject clientInfo = JObject.Parse(clientInfoJson);
        JArray redirectUris = clientInfo["redirect_uris"]?.ToObject<JArray>();

        // Check if the redirect URI is in the registered URIs
        if (redirectUris != null) {
            foreach (var uri in redirectUris) {
                // Normalize the URI from the cache for comparison
                string registeredUri = System.Net.WebUtility.UrlDecode(uri.ToString());
                if (registeredUri == redirectUri) {
                    return true;
                }
            }
        }

        context.Trace($"Redirect URI mismatch - URI: {redirectUri} not found in registered URIs");
        return false;
    }
    catch (Exception ex) {
        context.Trace($"Error checking client registration: {ex.Message}");
        return false;
    }
}" />
```

**何をしているか：**

-   `/register` で保存された `ClientInfo-{client_id}` をキャッシュから取得
-   クライアント ID の存在確認
-   リダイレクト URI の検証 - 登録された URI と一致するかチェック
-   未登録クライアントや URI ミスマッチを検出

### 4: クライアント情報の取得

```xml
<!-- Get client name and URI from cache -->
<set-variable name="client_name" value="@{
    try {
        string clientId = context.Variables.GetValueOrDefault<string>("client_id", "");

        if (string.IsNullOrEmpty(clientId)) {
            return "Unknown Application";
        }

        // Get the client info from the variable set by cache-lookup-value
        string clientInfoJson = context.Variables.GetValueOrDefault<string>("clientInfoJson");

        if (string.IsNullOrEmpty(clientInfoJson)) {
            return clientId; // Fall back to client ID if no name found
        }

        // Parse client info
        JObject clientInfo = JObject.Parse(clientInfoJson);
        string clientName = clientInfo["client_name"]?.ToString();

        return string.IsNullOrEmpty(clientName) ? clientId : clientName;
    }
    catch (Exception ex) {
        context.Trace($"Error retrieving client name: {ex.Message}");
        return context.Variables.GetValueOrDefault<string>("client_id", "Unknown Application");
    }
}" />
```

**何をしているか：**

-   同意画面に表示するためのクライアント名を取得
-   例：「MCP Inspector」など

### 5: 既存の同意状況の確認

```xml
<set-variable name="has_approval_cookie" value="@{
    try {
        string clientId = context.Variables.GetValueOrDefault<string>("client_id", "");
        string redirectUri = context.Variables.GetValueOrDefault<string>("normalized_redirect_uri", "");

        // Define a consistent cookie name for approvals
        string APPROVAL_COOKIE_NAME = "MCP_APPROVED_CLIENTS";

        // Check for cookie in request
        var cookieHeader = context.Request.Headers.GetValueOrDefault("Cookie", "");
        if (string.IsNullOrEmpty(cookieHeader)) {
            return false;
        }

        // Parse cookies
        string[] cookies = cookieHeader.Split(';');
        foreach (string cookie in cookies) {
            string trimmedCookie = cookie.Trim();
            if (trimmedCookie.StartsWith(APPROVAL_COOKIE_NAME + "=")) {
                // Extract the cookie value
                string cookieValue = trimmedCookie.Substring(APPROVAL_COOKIE_NAME.Length + 1);

                // Parse the base64-encoded JSON array of approved clients
                string decodedValue = System.Text.Encoding.UTF8.GetString(
                    System.Convert.FromBase64String(cookieValue.Split('.')[0]));
                JArray approvedClients = JArray.Parse(decodedValue);

                // Create a unique identifier for this client/redirect combination
                string clientKey = $"{clientId}:{redirectUri}";

                // Check if this client/redirect is in the approved list
                foreach (var item in approvedClients) {
                    if (item.ToString() == clientKey) {
                        return true;
                    }
                }
            }
        }
        return false;
    }
    catch (Exception ex) {
        context.Trace($"Error checking approval cookie: {ex.Message}");
        return false;
    }
}" />
```

**何をしているか：**

-   クッキー `MCP_APPROVED_CLIENTS` （同意済みクライアントのリスト（Base64 エンコードされた JSON 配列））を確認
-   クライアント ID + リダイレクト URI の組み合わせで管理
    -   例：`["abc123:http://localhost:8080/callback", "def456:http://localhost:3000/callback"]`

### 6: 同意画面の表示

```xml
<!-- Display consent page for unregistered clients -->
<choose>
    <when condition="@(!context.Variables.GetValueOrDefault<bool>("is_client_registered"))">
        <!-- Return client not found error page -->
    </when>
    <when condition="@(context.Variables.GetValueOrDefault<bool>("has_approval_cookie"))">
        <!-- Already approved - redirect back to authorize -->
    </when>
    <when condition="@(context.Variables.GetValueOrDefault<bool>("has_denial_cookie"))">
        <!-- Previously denied - show access denied page -->
    </when>
    <otherwise>
        <!-- Show consent page -->
    </otherwise>
</choose>
```

**同意画面の HTML 例（簡略版）：**

```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <title>Grant Access Permission</title>
        <style>
            /* モダンなスタイル */
        </style>
    </head>
    <body>
        <div class="consent-container">
            <h1 class="consent-heading">Grant Access Permission</h1>

            <p>
                The application <strong>Claude Desktop</strong> is requesting
                access to your MCP Server.
            </p>

            <div class="client-info">
                <p><strong>Application Name:</strong> Claude Desktop</p>
                <p><strong>Client ID:</strong> abc123</p>
                <p><strong>Requested Scopes:</strong> openid profile email</p>
            </div>

            <div class="buttons">
                <form method="post" style="display: inline;">
                    <input type="hidden" name="consent" value="allow" />
                    <input type="hidden" name="client_id" value="abc123" />
                    <input
                        type="hidden"
                        name="redirect_uri"
                        value="http://localhost:8080/callback"
                    />
                    <input type="hidden" name="state" value="random123" />
                    <button type="submit" class="btn btn-primary">Allow</button>
                </form>

                <form method="post" style="display: inline;">
                    <input type="hidden" name="consent" value="deny" />
                    <input type="hidden" name="client_id" value="abc123" />
                    <input
                        type="hidden"
                        name="redirect_uri"
                        value="http://localhost:8080/callback"
                    />
                    <input type="hidden" name="state" value="random123" />
                    <button type="submit" class="btn btn-secondary">
                        Deny
                    </button>
                </form>
            </div>
        </div>
    </body>
</html>
```

### 7: POST 処理（ユーザーの選択結果）

ユーザーが「Allow」または「Deny」ボタンを押すと、POST リクエストが送信され、同じポリシーで処理されます：

1. Allow 選択時:
    - 同意クッキーを設定
    - `/authorize?consent=granted&...` にリダイレクト
2. Deny 選択時:
    - 拒否クッキーを設定
    - アクセス拒否画面を表示

### ステップ ③ まとめ

-   ✅ **クライアント検証**: 登録済みクライアント ID とリダイレクト URI の妥当性確認
-   ✅ **同意画面生成**: 動的 HTML による視覚的な同意インターフェース提供
-   ✅ **同意状況の永続化**: Base64 エンコードされた JSON クッキーによる長期間記憶
-   ✅ **拒否処理対応**: 同意拒否時のアクセス拒否画面とクッキー管理

この同意プロセスが完了すると、ユーザーは元の `/authorize` エンドポイントに戻り、実際の OAuth 認証フローが継続されます。

## ステップ ④ 同意語の認証継続（GET /authorize?consent=granted）

`/consent` で「Allow」を選択すると、MCP クライアントは同じ `/authorize` エンドポイントにリダイレクトされますが、今度は `consent=granted` パラメータが追加されています。これにより、同意確認をスキップして実際の OAuth 認証フローに進みます。

**リダイレクトされる URL 例：**

```
GET /authorize?client_id=abc123&redirect_uri=http://localhost:8080/callback&state=random123&consent=granted
```

### 重要な違い：同意確認のスキップ

前回解説した `authorize.policy.xml` のセクション 3 を再度見てみましょう：

```xml
<choose>
    <when condition="@(
        context.Variables.GetValueOrDefault<string>("consentStatus") == "granted" ||
        context.Variables.GetValueOrDefault<string>("consentCookie") == "granted"
    )">
        <!-- Continue with normal flow - client is authorized -->
    </when>
    <otherwise>
        <!-- Redirect to consent page -->
    </otherwise>
</choose>
```

**今回の実行パターン：**

-   `consentStatus` に "granted" が設定されているため、`<when>` 条件が 真 になる
-   `<otherwise>` （同意ページへのリダイレクト）は実行されない
-   STEP 4 以降の OAuth 認証フローが継続実行され、Entra ID 認証ページへのリダイレクトが行われる

### Entra ID での認証

ユーザーは Microsoft Entra ID の認証画面に遷移し：

-   既にログイン済み → 即座にコールバック
-   未ログイン → ユーザー名・パスワード入力 → コールバック

認証完了後、Entra ID は /oauth-callback にリダイレクトします。

### フロー全体の状況

ここまでのフローをまとめると、以下のようになります（再掲）：

```mermaid
flowchart LR
    A["/authorize"] --> B{ユーザーは同意済み？}
    B -- はい --> C[Entra ID<br>認証画面]
    B -- いいえ --> D["/consent"]
    D --> A
```

最終的に Entra ID のログインが完了すると、APIM の `/oauth-callback` エンドポイントにリダイレクトされ、認証コードを受け取ります。

それでは、次のステップである `/oauth-callback` エンドポイントの処理を見ていきましょう。

## ステップ ⑤ Entra ID からのコールバック処理（`GET /oauth-callback`）

ユーザーが Entra ID で認証を完了すると、Entra ID は APIM の `/oauth-callback` エンドポイントにリダイレクトします。このエンドポイントでは、受け取った認証コードをアクセス トークンと交換し、MCP クライアント用の暗号化されたセッション キーを生成して MCP クライアントに返します。

https://github.com/Azure-Samples/remote-mcp-apim-functions-python/blob/main/infra/app/apim-oauth/oauth-callback.policy.xml

**受信するコールバック例：**

```
GET /oauth-callback?code=entra-auth-code&state=entra-state-guid&session_state=session123
```

### 1: Entra ID からのパラメータ抽出

```xml
<!-- STEP 1: Extract the authorization code and state from Entra ID callback -->
<set-variable name="authCode" value="@((string)context.Request.Url.Query.GetValueOrDefault("code", ""))" />
<set-variable name="entraState" value="@((string)context.Request.Url.Query.GetValueOrDefault("state", ""))" />
<set-variable name="sessionState" value="@((string)context.Request.Url.Query.GetValueOrDefault("session_state", ""))" />
```

**受信するパラメータ：**

-   `code`: Entra ID が発行した認証コード（短時間で期限切れ）
-   `state`: `/authorize` で生成した Entra ID 用の state（キャッシュキーとして使用）
-   `session_state`: Entra ID のセッション状態（オプション）

### 2: 保存された PKCE 情報の取得

```xml
<!-- STEP 2: Retrieve stored PKCE code verifier using the state parameter -->
<cache-lookup-value key="@("CodeVerifier-"+context.Variables.GetValueOrDefault("entraState", ""))" variable-name="codeVerifier" />
```

**何をしているか：**

-   `/authorize` で保存した `CodeVerifier-{entraState}` から `code_verifier` を取得
-   この `code_verifier` は、Entra ID へのトークン要求で必要（PKCE 検証用）

### 3: Entra ID トークン要求の準備

```xml
<!-- STEP 3: Set token request parameters -->
<set-variable name="codeChallengeMethod" value="S256" />
<set-variable name="redirectUri" value="{{OAuthCallbackUri}}" />
<set-variable name="clientId" value="{{EntraIDClientId}}" />
<set-variable name="clientAssertionType" value="@(System.Net.WebUtility.UrlEncode("urn:ietf:params:oauth:client-assertion-type:jwt-bearer"))" />
<authentication-managed-identity resource="api://AzureADTokenExchange" client-id="{{EntraIDFicClientId}}" output-token-variable-name="ficToken"/>
```

**authentication-managed-identity が何をしているか**

API Management (APIM) が割り当てられたマネージド ID を使って Entra ID からアクセス トークンを取得し、`Authorization: Bearer <token>` を自動で付与・キャッシュしてくれるポリシー です。

1. トークン取得処理を自動化
    - 指定した `client-id` に指定した `EntraIDFicClientId` は APIM に割り当てられたユーザー割り当てマネージド ID のクライアント ID です。これを使って Entra ID からトークンを取得します。
2. `resource` はトークンの宛先 (aud)
    - ここでは `api://AzureADTokenExchange` を指定。これは Federated Identity Credential (FIC) で指定されているトークン交換用 Audience です。
3. 取得した JWT を変数に保存
    - `output-token-variable-name="ficToken"` で保存し、後続の `<set-header>` や `<set-body>` から `@{context.Variables["ficToken"]}` として参照できます。

![登録アプリのフェデレーション情報](/images/mcp-entra-auth-using-apim/uami-fic.png)

### 4: Entra ID へのトークン要求

```xml
<!-- STEP 4: Configure token request to Entra ID -->
<set-method>POST</set-method>
<set-header name="Content-Type" exists-action="override">
    <value>application/x-www-form-urlencoded</value>
</set-header>
<set-body>@{
    return $"client_id={context.Variables.GetValueOrDefault("clientId")}&grant_type=authorization_code&code={context.Variables.GetValueOrDefault("authCode")}&redirect_uri={context.Variables.GetValueOrDefault("redirectUri")}&scope=https://graph.microsoft.com/.default&code_verifier={context.Variables.GetValueOrDefault("codeVerifier")}&client_assertion_type={context.Variables.GetValueOrDefault("clientAssertionType")}&client_assertion={context.Variables.GetValueOrDefault("ficToken")}";
}</set-body>
<rewrite-uri template="/token" />
```

**送信される実際のリクエスト例：**

```
POST https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

client_id=entra-app-client-id&
grant_type=authorization_code&
code=entra-auth-code&
redirect_uri=https://your-apim.azure-api.net/oauth-callback&
scope=https://graph.microsoft.com/.default&
code_verifier=generated-code-verifier&
client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3Aclient-assertion-type%3Ajwt-bearer&
client_assertion=jwt-token-from-managed-identity
```

**この段階で：**

-   APIM から Entra ID にバックエンド呼び出しが実行される
-   Entra ID がアクセス トークンとリフレッシュ トークンを返す

### 5: Entra ID 空のレスポンス処理

```xml
<!-- STEP 5: Process the token response from Entra ID -->
<trace source="apim-policy">
    <message>@("Token response received: " + context.Response.Body.As<string>(preserveContent: true))</message>
</trace>
```

**Entra ID からの典型的なレスポンス：**

```json
{
    "access_token": "eyJ0eXAiOiJKV1...",
    "token_type": "Bearer",
    "expires_in": 3600,
    "refresh_token": "0.AXoA...",
    "scope": "https://graph.microsoft.com/.default"
}
```

### 6: MCP クライアント用セッションキーの生成

```xml
<!-- STEP 6: Generate secure session token for MCP client -->
<set-variable name="IV" value="{{EncryptionIV}}" />
<set-variable name="key" value="{{EncryptionKey}}" />
<set-variable name="sessionId" value="@((string)Guid.NewGuid().ToString().Replace("-", ""))" />
<set-variable name="encryptedSessionKey" value="@{
    // Generate a unique session ID
    string sessionId = (string)context.Variables.GetValueOrDefault("sessionId");
    byte[] sessionIdBytes = Encoding.UTF8.GetBytes(sessionId);

    // Encrypt the session ID using AES
    byte[] IV = Convert.FromBase64String((string)context.Variables["IV"]);
    byte[] key = Convert.FromBase64String((string)context.Variables["key"]);

    byte[] encryptedBytes = sessionIdBytes.Encrypt("Aes", key, IV);

    return Convert.ToBase64String(encryptedBytes);
}" />
```

**暗号化プロセス：**

1. 新しいセッション ID（GUID）を生成
2. デプロイ時に生成された AES-256 キーと IV を使用
3. セッション ID を AES 暗号化して Base64 エンコード
4. この暗号化されたキーが MCP クライアントに返される「アクセス トークン」になる

### 7~9: MCP クライアント情報の復元

```xml
<!-- STEP 7: Lookup MCP client redirect URI stored during authorization -->
<cache-lookup-value key="@((string)context.Variables.GetValueOrDefault("entraState"))" variable-name="mcpConfirmConsentCode" />
<cache-lookup-value key="@($"McpClientAuthData-{context.Variables.GetValueOrDefault("mcpConfirmConsentCode")}")" variable-name="mcpClientData" />

<!-- STEP 8: Extract the stored mcp client state from cache -->
<set-variable name="mcpState" value="@{
    var mcpAuthDataAsJObject = JObject.Parse((string)context.Variables["mcpClientData"]);
    return (string)mcpAuthDataAsJObject["mcpClientState"];
}" />

<!-- STEP 9: Extract the stored mcp client callback redirect uri from cache -->
<set-variable name="callbackRedirectUri" value="@{
    var mcpAuthDataAsJObject = JObject.Parse((string)context.Variables["mcpClientData"]);
    return mcpAuthDataAsJObject["mcpCallbackRedirectUri"];
}" />
```

**復元される情報：**

-   MCP クライアントの元の state 値
-   MCP クライアントのコールバック URI（例：`http://localhost:8080/callback`）
-   元の PKCE 情報（後のトークン交換で検証用）

### 10: トークン情報のキャッシュ保存

```xml
<!-- STEP 10: Store the encrypted session key and Entra token in cache -->
<!-- Store the encrypted session key with the MCP confirmation code as key -->
<cache-store-value duration="3600"
    key="@($"AccessToken-{context.Variables.GetValueOrDefault("mcpConfirmConsentCode")}")"
    value="@($"{context.Variables.GetValueOrDefault("encryptedSessionKey")}")" />

<!-- Store the Entra token for later use -->
<cache-store-value duration="3600"
    key="@($"EntraToken-{context.Variables.GetValueOrDefault("sessionId")}")"
    value="@(context.Response.Body.As<JObject>(preserveContent: true).ToString())" />
```

**保存される情報：**

-   `AccessToken-{confirmCode}` → 暗号化されたセッション キー（MCP クライアントが後で使用）
-   `EntraToken-{sessionId}` → Entra ID の実際のアクセス トークン（MCP API 呼び出し時に検証用）

### 11: MCP クライアントへのリダイレクト

```xml
<!-- STEP 11: Redirect back to MCP client with confirmation code -->
<return-response>
    <set-status code="302" reason="Found" />
    <set-header name="Location" exists-action="override">
        <value>@($"{context.Variables.GetValueOrDefault("callbackRedirectUri")}?code={context.Variables.GetValueOrDefault("mcpConfirmConsentCode")}&state={context.Variables.GetValueOrDefault("mcpState")}&state_session=statesession123")</value>
    </set-header>
    <set-body />
</return-response>
```

**生成されるリダイレクト例：**

```
HTTP/1.1 302 Found
Location: http://localhost:8080/callback?code=mcp-confirm-code&state=original-mcp-state&state_session=statesession123
```

### ステップ ⑤ まとめ

-   ✅ **認証コード受信**: Entra ID からの認証コードと state 情報を抽出
-   ✅ **バックエンド トークン交換**: マネージド ID によるセキュアな Entra ID トークン取得
-   ✅ **セッションキー生成**: AES-256 暗号化による独自セッション管理システム構築
-   ✅ **データマッピング**: Entra ID トークンと暗号化セッションキーの関連付け

この段階で、MCP クライアントは確認コードを受け取り、次のステップである `/token` エンドポイントを呼び出してアクセス トークン（暗号化されたセッション キー）を取得する準備が整いました。

## ステップ ⑥ トークン交換（`POST /token`）

MCP クライアントは `/oauth-callback` から確認コードを受け取った後、`POST /token` エンドポイントを呼び出して実際のアクセス トークンを取得します。ここで PKCE 検証とクライアント登録確認が行われ、暗号化されたセッション キーがアクセス トークンとして返されます。

https://github.com/Azure-Samples/remote-mcp-apim-functions-python/blob/main/infra/app/apim-oauth/token.policy.xml

**MCP クライアントからのリクエスト例：**

```
POST /token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=mcp-confirm-code&
code_verifier=original-mcp-code-verifier&
client_id=registered-client-id&
redirect_uri=http://localhost:8080/callback
```

### 1~2: リクエスト パラメータの抽出

```xml
<!-- STEP 1: Extract parameters from token request -->
<set-variable name="tokenRequestBody" value="@((string)context.Request.Body.As<string>(preserveContent: true))" />

<!-- Extract the confirmation code from the request -->
<set-variable name="mcpConfirmConsentCode" value="@{
    var body = context.Variables.GetValueOrDefault<string>("tokenRequestBody");
    if (!string.IsNullOrEmpty(body))
    {
        var pairs = body.Split('&');
        foreach (var pair in pairs)
        {
            var keyValue = pair.Split('=');
            if (keyValue.Length == 2 && keyValue[0] == "code")
            {
                return keyValue[1];
            }
        }
    }
    return "";
}" />

<!-- Extract the code_verifier from the request and URL-decode it -->
<set-variable name="mcpClientCodeVerifier" value="@{
    // フォームデータからcode_verifierを抽出・URLデコード
    var body = context.Variables.GetValueOrDefault<string>("tokenRequestBody");
    // ... 同様の処理でcode_verifierを抽出
    return System.Net.WebUtility.UrlDecode(keyValue[1]);
}" />
```

**抽出される重要なパラメータ：**

-   `code`: `/oauth-callback` で発行された確認コード
-   `code_verifier`: MCP クライアントが最初に生成した PKCE 検証用の値

### 3: キャッシュから MCP クライアント認証データの取得

```xml
<!-- STEP 3: Retrieve stored MCP client data -->
<cache-lookup-value key="@($"McpClientAuthData-{context.Variables.GetValueOrDefault("mcpConfirmConsentCode")}")" variable-name="mcpClientAuthData" />

<!-- Extract the stored code challenge from the cached data -->
<set-variable name="storedMcpClientCodeChallenge" value="@{
    var mcpAuthDataAsJObject = JObject.Parse((string)context.Variables["mcpClientAuthData"]);
    return (string)mcpAuthDataAsJObject["mcpClientCodeChallenge"];
}" />
```

**取得される情報：**

-   `/authorize` で保存した MCP クライアントの元の `code_challenge`
-   その他の認証データ（state、scope、redirect_uri 等）

### 4: PKCE 検証の実行

```xml
<!-- STEP 4: Compute and validate the code challenge -->
<set-variable name="mcpServerComputedCodeChallenge" value="@{
    var mcpAuthDataAsJObject = JObject.Parse((string)context.Variables["mcpClientAuthData"]);
    string codeVerifier = (string)context.Variables.GetValueOrDefault("mcpClientCodeVerifier", "");
    string codeChallengeMethod = ((string)mcpAuthDataAsJObject["mcpClientCodeChallengeMethod"]).ToLower();

    if(codeChallengeMethod == "s256"){
        // For S256, compute the SHA256 hash, Base64 encode it, and convert to URL-safe format
        using (var sha256 = System.Security.Cryptography.SHA256.Create())
        {
            var bytes = System.Text.Encoding.UTF8.GetBytes(codeVerifier);
            var hash = sha256.ComputeHash(bytes);

            string base64 = Convert.ToBase64String(hash);
            // Convert Base64 string into a URL-safe variant
            return base64.Replace("+", "-").Replace("/", "_").Replace("=", "");
        }
    }
    return string.Empty;
}" />

<!-- STEP 5: Verify code challenge matches -->
<choose>
    <when condition="@(string.Compare((string)context.Variables.GetValueOrDefault("mcpServerComputedCodeChallenge", ""), (string)context.Variables.GetValueOrDefault("storedMcpClientCodeChallenge", "")) != 0)">
        <!-- If they don't match, return an error -->
        <return-response>
            <set-status code="400" reason="Bad Request" />
            <set-body>@("{\"error\": \"code_verifier does not match.\"}")</set-body>
        </return-response>
    </when>
</choose>
```

**PKCE 検証プロセス：**

1. 受信した `code_verifier` を SHA256 でハッシュ化
2. Base64URL エンコードして `code_challenge` を再計算
3. 保存されていた元の `code_challenge` と比較
4. 一致しない場合はエラーレスポンス（400 Bad Request）

### 5: クライアント登録確認

```xml
<!-- STEP 5.5: Verify client registration -->
<set-variable name="client_id" value="@{
    // フォームデータからclient_idを抽出
}" />
<set-variable name="redirect_uri" value="@{
    // フォームデータからredirect_uriを抽出・URLデコード
}" />

<!-- Look up client information from cache -->
<cache-lookup-value key="@($"ClientInfo-{context.Variables.GetValueOrDefault<string>("client_id")}")" variable-name="clientInfoJson" />

<!-- Verify that the client exists and the redirect URI is valid -->
<set-variable name="is_client_registered" value="@{
    try {
        string clientId = context.Variables.GetValueOrDefault<string>("client_id", "");
        string redirectUri = context.Variables.GetValueOrDefault<string>("normalized_redirect_uri", "");

        // Get the client info from the variable set by cache-lookup-value
        string clientInfoJson = context.Variables.GetValueOrDefault<string>("clientInfoJson");
        if (string.IsNullOrEmpty(clientInfoJson)) {
            return false;
        }

        // Parse client info
        JObject clientInfo = JObject.Parse(clientInfoJson);
        JArray redirectUris = clientInfo["redirect_uris"]?.ToObject<JArray>();

        // Check if the redirect URI is in the registered URIs
        if (redirectUris != null) {
            foreach (var uri in redirectUris) {
                string registeredUri = System.Net.WebUtility.UrlDecode(uri.ToString());
                if (registeredUri == redirectUri) {
                    return true;
                }
            }
        }
        return false;
    }
    catch (Exception ex) {
        return false;
    }
}" />

<choose>
    <when condition="@(!context.Variables.GetValueOrDefault<bool>("is_client_registered"))">
        <!-- Client is not properly registered, return error -->
        <return-response>
            <set-status code="401" reason="Unauthorized" />
            <set-header name="Content-Type" exists-action="override">
                <value>application/json</value>
            </set-header>
            <set-body>@{
                var errorResponse = new JObject();
                errorResponse["error"] = "invalid_client";
                errorResponse["error_description"] = "Client not found or redirect URI is invalid.";
                return errorResponse.ToString();
            }</set-body>
        </return-response>
    </when>
</choose>
```

**セキュリティ検証：**

-   クライアント ID が `/register` で登録されているか確認
-   リダイレクト URI が登録された URI と一致するか確認
-   未登録または不一致の場合は 401 Unauthorized エラー

### 6: 暗号化されたセッションキーの取得と返却

```xml
<!-- STEP 6: Retrieve cached tokens -->
<cache-lookup-value key="@($"AccessToken-{context.Variables.GetValueOrDefault("mcpConfirmConsentCode")}")" variable-name="cachedSessionToken" />

<!-- STEP 7: Generate token response -->
<set-variable name="jsonPayload" value="@{
    var accessToken = context.Variables.GetValueOrDefault<string>("cachedSessionToken");
    var payloadObject = new
    {
        access_token = accessToken,
        token_type = "Bearer",
        expires_in = 3600,
        refresh_token = "",
        scope = "openid profile email"
    };

    return Newtonsoft.Json.JsonConvert.SerializeObject(payloadObject);
}" />
<set-body template="none">@{
    return (string)context.Variables.GetValueOrDefault("jsonPayload", "");
}</set-body>
```

**MCP クライアントに返されるレスポンス例：**

```json
{
    "access_token": "eyJhbGciOiJBMTI4R0NNIiwi...（暗号化されたセッションキー）",
    "token_type": "Bearer",
    "expires_in": 3600,
    "refresh_token": "",
    "scope": "openid profile email"
}
```

**重要なポイント：**

-   `access_token` は実際の Entra ID トークンではなく、APIM 独自の暗号化されたセッションキー
-   このトークンを使って MCP API にアクセスする際、APIM が復号化して Entra ID トークンと照合

### ステップ ⑥ まとめ

-   ✅ **PKCE 検証**: code_verifier から code_challenge を再計算して元の値と照合
-   ✅ **クライアント検証**: 登録済みクライアントとリダイレクト URI の妥当性確認
-   ✅ **トークン交換**: 確認コードを暗号化されたセッションキーと交換
-   ✅ **CORS 対応**: クロスオリジンアクセス許可ヘッダー設定

この段階で、MCP クライアントは暗号化されたアクセストークンを取得し、MCP API の呼び出しが可能になります。次のステップは実際の MCP API（`/mcp/sse` や`/mcp/message`）へのアクセスになります。

## ステップ ⑦ MCP API アクセス（`GET /mcp/sse`, `POST /mcp/message`）

MCP クライアントが `/token` から暗号化されたアクセス トークンを取得した後、実際の MCP API（`/mcp/sse` や `/mcp/message`）にアクセスします。このエンドポイントでは、暗号化されたセッション キーの復号化、Entra ID トークンの検証、そして Azure Functions への安全なプロキシが行われます。

https://github.com/Azure-Samples/remote-mcp-apim-functions-python/blob/main/infra/app/apim-mcp/mcp-api.policy.xml

**MCP クライアントからのリクエスト例：**

```
GET /mcp/sse
Authorization: Bearer eyJhbGciOiJBMTI4R0NNIiwi...（暗号化されたセッション キー）

POST /mcp/message
Authorization: Bearer eyJhbGciOiJBMTI4R0NNIiwi...（暗号化されたセッション キー）
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "method": "tools/list",
  "id": 1
}
```

### 1: Authorization ヘッダーの検証

```xml
<check-header name="Authorization" failed-check-httpcode="401" failed-check-error-message="Not authorized" ignore-case="false" />
```

**何をしているか：**

-   Authorization ヘッダーの存在を確認
-   存在しない場合は即座に 401 エラーを返す

### 2: 暗号化されたセッションキーの復号化

```xml
<set-variable name="IV" value="{{EncryptionIV}}" />
<set-variable name="key" value="{{EncryptionKey}}" />
<set-variable name="decryptedSessionKey" value="@{
    // Retrieve the encrypted session key from the request header
    string authHeader = context.Request.Headers.GetValueOrDefault("Authorization");

    string encryptedSessionKey = authHeader.StartsWith("Bearer ") ? authHeader.Substring(7) : authHeader;

    // Decrypt the session key using AES
    byte[] IV = Convert.FromBase64String((string)context.Variables["IV"]);
    byte[] key = Convert.FromBase64String((string)context.Variables["key"]);

    byte[] encryptedBytes = Convert.FromBase64String(encryptedSessionKey);
    byte[] decryptedBytes = encryptedBytes.Decrypt("Aes", key, IV);

    return Encoding.UTF8.GetString(decryptedBytes);
}" />
```

**復号化プロセス：**

-   `Authorization: Bearer xxx` からトークン部分を抽出
-   デプロイ時に設定された AES-256 キーと IV を取得
-   Base64 デコードしてから AES 復号化を実行
-   復号化されたセッション ID（GUID 形式）を取得

### 3: Entra ID トークンの検索と検証

```xml
<cache-lookup-value key="@($"EntraToken-{context.Variables.GetValueOrDefault("decryptedSessionKey")}")" variable-name="accessToken" />

<choose>
    <when condition="@(context.Variables.GetValueOrDefault("accessToken") == null)">
        <return-response>
            <set-status code="401" reason="Unauthorized" />
            <set-header name="WWW-Authenticate" exists-action="override">
                <value>Bearer error="invalid_token"</value>
            </set-header>
        </return-response>
    </when>
</choose>
```

**検証プロセス：**

-   復号化されたセッション ID をキーとして `EntraToken-{sessionId}` をキャッシュ検索
-   対応する Entra ID トークンが存在しない場合は 401 エラー
-   これにより、有効なセッションのみが MCP API にアクセス可能

### 4: Azure Functions への安全なプロキシ

```xml
<set-header name="x-functions-key" exists-action="override">
    <value>{{function-host-key}}</value>
</set-header>
```

**プロキシ設定：**

-   Azure Functions のホストキーを HTTP ヘッダーに追加
-   リクエストがそのまま Azure Functions（MCP サーバー実装）に転送される
-   レスポンスもそのまま MCP クライアントに返される

**セキュリティアーキテクチャの全体像**

```
MCPクライアント
    ↓ Authorization: Bearer {暗号化セッションキー}
APIM (mcp-api.policy.xml)
    ↓ 1. セッションキー復号化
    ↓ 2. Entra IDトークン検証
    ↓ 3. x-functions-key 追加
Azure Functions (MCPサーバー)
    ↓ レスポンス
MCPクライアント
```

**エラーハンドリング**

無効なトークンの場合:

```
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer error="invalid_token"
```

期限切れセッションの場合:

```
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer error="invalid_token"
```

**期限切れの仕組み：**

-   キャッシュエントリは 3600 秒（1 時間）で自動削除
-   期限切れ後は `cache-lookup-value` が null を返すため自動的に認証失敗

### ステップ ⑦ まとめ

-   ✅ **トークン復号化**: AES-256 でセッションキーを安全に復号化
-   ✅ **セッション検証**: 有効な Entra ID トークンとの紐付け確認
-   ✅ **プロキシ実行**: Azure Functions への安全なリクエスト転送
-   ✅ **自動期限管理**: キャッシュベースの自動セッション期限切れ

この段階で、MCP クライアントは完全に認証され、Azure Functions 上の MCP サーバーの機能（ツール実行、リソース取得等）にアクセスできるようになります。APIM は透明なプロキシとして動作し、バックエンドの MCP サーバーは認証を意識することなく、純粋な MCP プロトコルの実装に集中できます。

# シーケンス図

以下は、ここまでのフローをシーケンス図で表現したものです。各ステップのやり取りがどのように行われるかを視覚的に示しています。

```mermaid
sequenceDiagram
    participant MCP as MCPクライアント<br/>(Claude等)
    participant APIM as Azure API Management
    participant Entra as Microsoft Entra ID
    participant Function as Azure Functions<br/>(MCPサーバー)

    Note over MCP,Function: ステップ1: クライアント登録
    MCP->>APIM: POST /register<br/>client_name, redirect_uris
    APIM->>APIM: クライアントID生成<br/>キャッシュに保存
    APIM-->>MCP: client_id, redirect_uris

    Note over MCP,Function: ステップ2: 認証開始
    MCP->>APIM: GET /authorize<br/>client_id, code_challenge, redirect_uri
    APIM->>APIM: 同意確認(クッキー)

    alt 未同意の場合
        APIM-->>MCP: 302 → /consent
        Note over MCP,Function: ステップ3: ユーザー同意
        MCP->>APIM: GET /consent<br/>同意画面表示
        APIM-->>MCP: HTML同意画面
        MCP->>APIM: POST /consent<br/>consent_action=allow
        APIM->>APIM: 同意クッキー設定
        APIM-->>MCP: 302 → /authorize?consent=granted
    end

    Note over MCP,Function: ステップ4: 認証継続
    MCP->>APIM: GET /authorize?consent=granted
    APIM->>APIM: Entra ID用PKCE生成<br/>認証データをキャッシュ
    APIM-->>MCP: 302 → Entra ID認証画面

    Note over MCP,Function: ステップ5: Entra ID認証
    MCP->>Entra: GET /authorize<br/>Entra ID認証画面
    Entra-->>MCP: ユーザー認証完了
    Entra->>APIM: GET /oauth-callback<br/>code, state

    Note over MCP,Function: ステップ6: トークン交換(APIM⇔Entra)
    APIM->>APIM: code_verifier取得
    APIM->>Entra: POST /token<br/>code, code_verifier, client_assertion
    Entra-->>APIM: access_token, refresh_token
    APIM->>APIM: セッションキー暗号化<br/>トークンをキャッシュ
    APIM-->>MCP: 302 → MCPクライアントコールバック

    Note over MCP,Function: ステップ7: トークン取得
    MCP->>APIM: POST /token<br/>code, code_verifier, client_id
    APIM->>APIM: PKCE検証<br/>クライアント検証
    APIM->>APIM: 暗号化セッションキー取得
    APIM-->>MCP: 暗号化されたaccess_token

    Note over MCP,Function: ステップ8: MCP API呼び出し
    MCP->>APIM: GET /mcp/sse<br/>Authorization: Bearer {暗号化トークン}
    APIM->>APIM: トークン復号化<br/>Entraトークン検証
    APIM->>Function: x-functions-key付きでプロキシ
    Function-->>APIM: MCPレスポンス
    APIM-->>MCP: MCPレスポンス

    MCP->>APIM: POST /mcp/message<br/>Authorization: Bearer {暗号化トークン}
    APIM->>APIM: トークン復号化<br/>Entraトークン検証
    APIM->>Function: x-functions-key付きでプロキシ
    Function-->>APIM: MCPレスポンス
    APIM-->>MCP: MCPレスポンス
```

# まとめ

この記事では、APIM を使用して Entra ID 認証を行い、MCP クライアントが安全に Azure Functions 上の MCP サーバーにアクセスするための一連のフローを詳しく解説しました。

APIM を認証・認可のゲートウェイとして利用することで、**MCP サーバー側は認証・認可の実装から分離され、純粋な MCP プロトコルの実装に集中できます。**また、APIM のポリシーを活用することで、セキュリティやトークン管理の複雑さを大幅に軽減できます。
