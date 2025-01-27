---
title: "【azurenv】ローカルの環境変数を Azure アプリへ同期させる CLI ツール"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "cli", "env"]
published: true
publication_name: "microsoft"
---

こんにちは、モモスケです。
今回は、ローカルの `.env` ファイルから **Azure App Service** や **Azure Functions** 上の環境変数を一括で更新できる CLI ツール 「**azurenv**」 を開発したのでご紹介します。

# azurenv とは？

**azurenv** は、Go 言語で実装された **クロスプラットフォーム CLI ツール** です。
手元の `.env` ファイルをコピー＆ペーストして Azure Portal に貼り付けるような作業を **1 コマンド** で自動化できるのが大きな特徴です。

![](/images/sync-app-settings/azurenv-demo.gif)

## 主な機能

-   App Service や Azure Functions に既に登録されてる環境変数を取得し表示
-   ローカル ファイルと Azure の環境変数を比較し、差分を表示
-   **ローカル ファイル（例：`.env`）を読み込み、Azure に環境変数を一括更新**

:::message
**注意: この CLI は Azure CLI (az) に依存しています。必ず 最新の Azure CLI をインストール＆ログイン（`az login`）しておいてください。**
:::

# 背景

**App Service** や **Azure Functions** には、環境変数を設定することができます。

その際、ローカルで開発している際には `.env` などのファイルに環境変数を記述し、それを Azure Portal にコピー＆ペーストすることが一般的です。

1. .env ファイルに環境変数を定義
2. Azure Portal で「アプリケーション設定」を開いて、手動でコピペ or 手打ち
3. 値が増減するたびに、また同じ作業を繰り返す…

とても煩雑で、**キーを打ち間違えてしまうリスク**も大きいです。
**azurenv** はこの手間をなくすことを目的に開発しました。

::: message
Azure Functions 開発において、環境変数を `local.settings.json` に記述している場合は、`azurenv` を使わずに以下のコマンドでローカル環境変数を Azure Functions に反映できます。

```bash
func azure functionapp publish myfunctionapp12345 --publish-settings-only
```

`func azure functionapp publish` コマンドは Azure Functions をデプロイするものですが、`--publish-settings-only` フラグを付けることで環境変数のみを更新することができます。

また、デプロイと同時に環境変数を更新する場合は、`--publish-local-settings` フラグを使います。

[Core Tools を使用してローカルで Azure Functions を開発する](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-run-local?tabs=windows%2Cisolated-process%2Cnode-v4%2Cpython-v2%2Chttp-trigger%2Ccontainer-apps&pivots=programming-language-python#upload-local-settings-to-azure)
:::

# インストール方法

## macOS / Linux

```bash
curl -fsSL https://raw.githubusercontent.com/ry0y4n/azurenv/main/scripts/install.sh | sh
```

## Windows

```powershell
powershell -Command "Invoke-WebRequest -Uri https://raw.githubusercontent.com/ry0y4n/azurenv/main/scripts/install.ps1 -OutFile install.ps1; .\install.ps1"
```

詳細は以下の GitHub リポジトリを参照してください。
https://github.com/ry0y4n/azurenv

# コマンド一覧と使い方

## App Service 関連コマンド

### 1. `azurenv webapp list-remote`

App Service のアプリ設定を取得します。

```bash
azurenv webapp list-remote --name <WebAppName> --resource-group <ResourceGroupName>
```

### 2. `azurenv webapp diff`

ローカルの `.env` と App Service のアプリ設定を比較し、変更差分を確認できます。

```bash
azurenv webapp diff --name <WebAppName> --resource-group <ResourceGroupName> --file <.env>
```

### 3. `azurenv webapp apply`

ローカルの `.env` を App Service のアプリ設定に反映します。

```bash
azurenv webapp apply --name <WebAppName> --resource-group <ResourceGroupName> --file <.env>
```

## Azure Functions 関連コマンド

### 1. `azurenv function list-remote`

Azure Functions のアプリ設定を取得します。

```bash
azurenv functionapp list-remote --name <FunctionAppName> --resource-group <ResourceGroupName>
```

### 2. `azurenv function diff`

ローカルの `.env` と Azure Functions のアプリ設定を比較します。

```bash
azurenv functionapp diff --name <FunctionAppName> --resource-group <ResourceGroupName> --file <.env>
```

### 3. `azurenv function apply`

ローカルの `.env` を Azure Functions に一括で反映します。

```bash
azurenv functionapp apply --name <FunctionAppName> --resource-group <ResourceGroupName> --file <.env>
```

## その他のコマンド

-   `azurenv version`: バージョン情報を表示します。
-   `azurenv azcheck`: 現在ログインしている Azure CLI のアカウント情報を表示します。自分がどのサブスクリプションを使っているか手軽に確認出来て便利です。

## 補足

::: message
全ての **`--file`** フラグには任意のファイルパスを指定可能です（例：`./src/internal/.env.development`）。
フラグを指定しない場合、デフォルトはカレントディレクトリの `.env` を読み込みます。
:::

::: message
上書きや追加だけが行われ、リモートにあるキーを削除はしません。削除も行いたい場合は別途検討が必要です。
:::

# まとめ

-   **azurenv** を使うと、ローカルの `.env` を**1 コマンド**で App Service や Azure Functions に反映可能です。
-   手動コピペを繰り返す必要が無くなるため、**ヒューマンエラーの軽減 & 作業効率の向上**が期待できます。
-   事前に **Azure CLI (`az`)** が正しくインストール & ログインされているかを確認してください。
-   Go 言語で実装されているため、**クロスプラットフォーム**で依存関係無く動作するシングルバイナリとして配布されています。

バグや改善提案があれば、GitHub の Issue / PR でフィードバックをお寄せいただけると嬉しいです。
少しでも皆さんの **Azure 環境変数管理** がラクになることを願っています！

# 参考資料

-   [環境変数を登録する Azure CLI コマンド（App Service）](https://learn.microsoft.com/ja-jp/cli/azure/webapp/config/appsettings?view=azure-cli-latest)
-   [環境変数を登録する Azure CLI コマンド（Azure Functions）](https://learn.microsoft.com/ja-jp/cli/azure/functionapp/config/appsettings?view=azure-cli-latest)
-   [【Envlate】.env ファイルから .env.template ファイルを生成するクロスプラットフォーム CLI ツール](https://zenn.dev/momosuke/articles/generate-env-template-cli)
