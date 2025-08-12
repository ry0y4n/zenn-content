---
title: "APIM のポリシー群を自動管理する GitHub Actions を公開しました"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure", "azureapimanagement", "githubactions", "api"]
published: true
publication_name: "microsoft"
---

## はじめに

**Azure API Management (APIM) のポリシー更新を、GitHub Actions で自動化するアクションを公開しました。** 社内の手続きを経て Azure 公式アクションとしても公開できました。この記事では、APIM 初心者の方にもわかるように、背景 → 使い方 → 運用のコツまでをサクッと解説します。

https://github.com/marketplace/actions/azure-api-management-policy-update

![Azure APIM + GitHub Actions のイメージ](/images/release-apim-policy-update-action/gha-market-view.png)

- アクション本体：[Azure API Management Policy Update](https://github.com/marketplace/actions/azure-api-management-policy-update)
- ラボ：[Azure API Management Policy Management Lab](https://github.com/ry0y4n/apim-policy-cicd-lab)

## APIM と「ポリシー」の超入門

- APIM は、API の入口に立つゲートウェイでありプロキシのような存在です。**認証・レート制限・変換・監視** などの横断機能を担います。
- ポリシーは、このゲートウェイのふるまいを XML で宣言する仕組みです。API 全体や API 内の各操作単位などに適用できます。

**例 ①：レート制限**

サブスクリプションあたりの呼び出しレートを 60 秒あたり 20 回に制限するポリシー

```xml
<policies>
    <inbound>
        <base />
        <rate-limit calls="20" renewal-period="60" remaining-calls-variable-name="remainingCallsPerSubscription"/>
    </inbound>
    <outbound>
        <base />
    </outbound>
</policies>
```

**例 ②：Azure OpenAI トークン制限**

呼び出し元 IP アドレスごとに、1 分あたり 5000 トークンの制限をかけるポリシー

```xml
<policies>
    <inbound>
        <base />
        <azure-openai-token-limit
            counter-key="@(context.Request.IpAddress)"
            tokens-per-minute="5000" estimate-prompt-tokens="false" remaining-tokens-variable-name="remainingTokens" />
    </inbound>
    <outbound>
        <base />
    </outbound>
</policies>
```

**例 ③：IP アドレス制限**

特定の IP アドレスまたはアドレス範囲からのアクセスのみを許可するポリシー

```xml
<ip-filter action="allow">
    <address>13.66.201.169</address>
    <address-range from="13.66.140.128" to="13.66.140.143" />
</ip-filter>
```

---

これらのように「ポリシーをコードとして管理」できるようになると、差分レビューや再現性の面で運用がかなり楽になります。

:::message
ポリシーの一覧は **[Azure API Management ポリシーのドキュメント](https://learn.microsoft.com/ja-jp/azure/api-management/api-management-policies)** で確認できます。
:::

## As-Is と To-Be（このアクションが解く課題）

- As-Is（よくある現状）
  - Azure ポータルでの手作業更新はミスが起きやすく、履歴も追いづらい
  - スクリプト自作は保守負担が大きい、ARM/Bicep は初期学習コストが重い
  - ポリシーの差分レビューや再現が難しく、属人化しがち
- To-Be（このアクションが目指す状態）
  - ポリシーを Git で管理し、PR で差分レビュー → マージで CI が自動適用
  - ポリシー フォルダ構成 or マニフェストで自動検出して更新
  - リソース存在チェックと基本 XML 検証、明確なログ、最後の ETag を出力
  - シンプル導入・小さく始めやすい更新フロー

## このアクションでできること（機能概要）

- 自動ポリシー発見
  - `policies/<apiId>/api.xml` と `policies/<apiId>/operations/<operationId>.xml`
  - またはマニフェスト `policy_manifest.yaml` で任意のポリシー ファイルのパスをマッピング
- リソースチェック: API や操作が存在するかを確認してから更新
- 基本 XML 検証: `<policies>…</policies>` ルートがあるかを確認
- ログとエラー: 失敗時に API 一覧や一部操作を出して文脈を付与
- 出力: 最後に Azure から返った`ETag` を出力
- 既知の非対応: 条件付き更新（If-Match）は未実装。差分判断は Git/PR で行う設計（早めに実装します。。。）

## クイックスタート（最短で導入）

以下のワークフローで、`/policies` 配下や `policy_manifest.yaml` の変更をトリガーにして、ポリシーの自動更新を行えます。

```yaml
name: Update APIM Policies

on:
  push:
    branches: [main]
    paths: ["policies/**", "policy_manifest.yaml"]
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  update-policies:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Update APIM Policies
        uses: Azure/apim-policy-update@v0.1.0
        id: update-apim-policy
        with:
          apim_name: ${{ secrets.AZURE_APIM_NAME }}
          resource_group: ${{ secrets.AZURE_RESOURCE_GROUP }}
          subscription_id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          policy_manifest_path: "policy_manifest.yaml"

      - run: echo "Last ETag=${{ steps.update-apim-policy.outputs.etag }}"
```

- 必要権限とシークレット
  - permissions: `id-token: write`（OIDC 推奨）, `contents: read`
  - シークレット: `AZURE_CLIENT_ID` / `AZURE_TENANT_ID` / `AZURE_SUBSCRIPTION_ID` / `AZURE_APIM_NAME` / `AZURE_RESOURCE_GROUP`
- 運用ヒント
  - `paths`で適用対象の変更だけをトリガー
  - main マージ時に適用、PR で差分レビューという分業がわかりやすい

実際にこのワークフローを動作させるためのラボは以下のリポジトリにあります。サンプルのポリシーやマニフェストも含まれていますので、ぜひ試してみてください。

https://github.com/ry0y4n/apim-policy-cicd-lab

## ポリシー配置の選び方（Convention vs Manifest）

- デフォルト: 初期はこれが簡単

```
policies/
├── sample-api/
│   ├── api.xml
│   └── operations/
│       ├── get-users.xml
│       └── post-users.xml
└── another-api/
    ├── api.xml
    └── operations/
        └── create-order.xml
```

- マニフェスト: 既存配置を崩したくない時や、柔軟に置きたい時は、以下のようなマニフェスト ファイルを配置して、アクションのインプットにパスを指定します。

```yaml
policies:
  users-api:
    apiPolicyPath: policies/users-api/api.xml
    operations:
      get-users: policies/users-api/operations/get-users.xml
      post-users: policies/users-api/operations/post-users.xml

  orders-api:
    apiPolicyPath: policies/orders-api/api.xml
    operations:
      create-order: policies/orders-api/operations/create-order.xml
```

判断基準の目安

- **新規/小規模**: デフォルトを採用し、慣れてきたらマニフェストに切り替えでも OK
- **既存資産がある**: まずマニフェストでマッピング → 段階的に整理

## 使い方の詳細（Inputs/Outputs/エラー）

### 入力パラメータ

| パラメータ             | 必須/任意 | 説明                                           |
| ---------------------- | --------- | ---------------------------------------------- |
| `subscription_id`      | 必須      | Azure サブスクリプション ID                    |
| `resource_group`       | 必須      | リソースグループ名                             |
| `apim_name`            | 必須      | APIM のサービス名                              |
| `policy_manifest_path` | 任意      | マニフェストのパス（省略時はデフォルトで探索） |

### 出力パラメータ

| パラメータ | 説明                      |
| ---------- | ------------------------- |
| `etag`     | 最後に成功した更新の ETag |

### 典型エラーと対処

- API/Operation が見つからない: ID の食い違いを確認。ログに API 一覧/一部操作が出るので照合する
- 認証/接続の失敗: OIDC の権限（`id-token: write`）とロール割り当てを確認
- XML 不備: `<policies>`ルートがない/閉じタグ漏れなど。CI で早めに検知
- デバッグを詳しく: `ACTIONS_STEP_DEBUG=true` をリポジトリシークレットに設定すると詳細ログ

## 既知の制限と今後

- If-Match による条件更新や差分適用は未実装
- XML スキーマの厳密検証は未対応（基本的なルート検証のみ）
- フィードバック募集中: どのユースケースで機能拡張があると嬉しいか教えてください

## おわりに

Git でポリシーを管理し、PR でレビューして、CI で安全に APIM へ反映する——この流れを最小コストで始められるよう設計しました。ぜひチームの運用に取り入れてみてください。
