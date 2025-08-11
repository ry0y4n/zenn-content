---
title: "Azure Login Action で OIDC 認証する最短手順（GitHub Actions）"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure", "githubactions", "github", "devops"]
published: true
pupublication_name: "microsoft"
---

GitHub Actions の [Azure Login Action (azure/login)](https://github.com/marketplace/actions/azure-login) を、推奨の OpenID Connect（OIDC）で安全にサインインさせるための最短手順をまとめます。シークレットにパスワードを持たないため、セキュアで運用しやすいのが利点です。

https://github.com/marketplace/actions/azure-login

## ゴール

- GitHub Actions から OIDC で Azure にサインインできる

## 前提

- Azure CLI が使えること
- 権限: 対象サブスクリプションでロール割り当てができること（「[所有者](https://learn.microsoft.com/ja-jp/azure/role-based-access-control/built-in-roles/privileged#owner)」または「[ロール ベースのアクセスの制御の管理者](https://learn.microsoft.com/ja-jp/azure/role-based-access-control/built-in-roles/privileged#role-based-access-control-administrator)」）

## 1) Microsoft Entra アプリとフェデレーション資格情報の準備

1. Azure にログイン

```bash
az login
```

2. アプリ登録（アプリケーション ID を控える）

```bash
az ad app create --display-name "<アプリ名>"
```

https://learn.microsoft.com/ja-jp/cli/azure/ad/app?view=azure-cli-latest#az-ad-app-create

出力の `appId` が「アプリケーション ID（Client ID）」です。

3. サービス プリンシパルを作成

```bash
az ad sp create --id <アプリケーション ID>
```

https://learn.microsoft.com/ja-jp/cli/azure/ad/sp?view=azure-cli-latest#az-ad-sp-create

4. GitHub Actions を信頼するフェデレーション資格情報を作成

```bash
az ad app federated-credential create \
  --id <アプリケーション ID> \
  --parameters '{
    "name": "<FEDERATED_CREDENTIAL_NAME>",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:<OWNER>/<REPOSITORY>:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

https://learn.microsoft.com/ja-jp/cli/azure/ad/app/federated-credential?view=azure-cli-latest#az-ad-app-federated-credential-create

主な subject 例（必要に応じて置き換え）:

- ブランチ固定: `repo:OWNER/REPO:ref:refs/heads/main`
- タグ固定: `repo:OWNER/REPO:ref:refs/tags/v1`
- 環境固定: `repo:OWNER/REPO:environment:production`
- PR 全体: `repo:OWNER/REPO:pull_request`

> 今回は、ブランチ固定で、`main` ブランチを指定しています。

https://learn.microsoft.com/ja-jp/entra/workload-id/workload-identity-federation-create-trust?pivots=identity-wif-apps-methods-azcli#github-actions-example

5. 最小権限ロールを割り当て（例: サブスクリプションに Contributor）

```bash
az role assignment create \
  --role contributor \
  --assignee <アプリケーション ID> \
  --scope /subscriptions/<サブスクリプション ID>
```

https://learn.microsoft.com/ja-jp/cli/azure/role/assignment?view=azure-cli-latest#az-role-assignment-create

必要に応じてスコープをリソース グループや特定リソースに絞ると、より最小権限になります。

## 2) GitHub シークレットの登録

リポジトリの Settings > Secrets and variables > Actions で以下 3 つを登録します。

- `AZURE_CLIENT_ID`: 上記アプリのアプリケーション ID（Client ID）
- `AZURE_TENANT_ID`: Entra テナント ID（Directory/Tenant ID）
- `AZURE_SUBSCRIPTION_ID`: サブスクリプション ID

## 3) ワークフロー例（azure/login@v2 + OIDC）

`id-token: write` を必ず付与します。`contents: read` はデフォルトで十分ですが、明示しておくと安心です。

```yaml
name: Azure OIDC Login sample

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  demo:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Azure login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Azure CLI - whoami
        uses: azure/cli@v2
        with:
          inlineScript: |
            az account show --output table
```

ワークフローを実行すると、以下のように Azure にログインできていることが確認できます。

![ワークフローの実行結果](/images/gha-azure-login-oidc/gha-workflow-result.png)

以降は [`azure/cli`](https://github.com/marketplace/actions/azure-cli-action) や各種 Azure Actions（Bicep/ARM/Storage/App Service など）でのデプロイに進めます。

## よくあるハマりどころ

- `id-token` の付与漏れ: `permissions: id-token: write` を忘れると OIDC トークンが発行されません。
- `subject` の不一致: 付与した `subject` がワークフローのトリガーと合っているか確認（ブランチ/タグ/環境）。

## 宣伝

最近、API Management のポリシーを自動更新する Azure 公式の GitHub Action を公開しました。これを使うと、API Management のポリシーを簡単に更新できますので、興味があればぜひ試してみてください。

https://github.com/marketplace/actions/azure-api-management-policy-update

## 参考

- Azure Login action: https://github.com/marketplace/actions/azure-login
- OIDC と GitHub Actions の連携（フェデレーション資格情報）: https://learn.microsoft.com/entra/workload-id/workload-identity-federation-create-trust
- Bicep 例（GitHub Actions での認証解説含む）: https://learn.microsoft.com/azure/azure-resource-manager/bicep/deploy-github-actions
