---
title: ""
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# 既存の MCP サーバーを最小限の変更で Azure Functions にホストしてみよう

## はじめに

Model Context Protocol（MCP）は、Anthropic が開発した AI エージェントとツール間の標準化されたプロト（以下略...!!）

これまでも Azure Functions で MCP サーバーを開発することは可能でしたが、それは Azure Functions のライブラリを使って**最初から Azure Functions での動作を前提として**開発する必要がありました。

しかし、多くの開発者は既にローカル環境で動作する MCP サーバーを構築済みで、「このサーバーをクラウドで動かしたいけど、一から書き直すのは面倒だな...」と感じているのではないでしょうか？

そんな要望に応えるために、**既存の MCP サーバーを最小限の変更で Azure Functions にホストできる**公式サンプルが Microsoft からリリースされました！

https://github.com/Azure-Samples/mcp-sdk-functions-hosting-python

このサンプルの最大の魅力は、**Azure Functions を全く意識せずに開発した既存の MCP サーバーを、ほんの少しの設定ファイル追加だけでクラウドにホストできる**ことです。

この記事では、実際にこのサンプルを使って以下の流れを体験していきます：

1. **MCP サーバーの作成** - Azure を意識せずにシンプルな MCP サーバーを作る
2. **Azure Functions 対応** - 設定ファイルを追加するだけで既存の MCP サーバーを Azure Functions 対応に
3. **ローカル実行** - `func start`でローカルで Azure Functions 環境を再現
4. **クラウドデプロイ** - 実際に Azure にデプロイしてリモートで動作確認

## 今回作るもの

今回は学習目的で、シンプルな**計算ツール**を提供する MCP サーバーを作成します。以下の機能を持ちます：

- 四則演算
  - 足し算
  - 引き算
  - 掛け算
  - 割り算

## 前提条件

この記事を進めるために、以下が必要です：

### 必須

- Python 3.8 以上
- Visual Studio Code
- Azure アカウント（無料アカウントで OK）

### インストールが必要なツール

- [Azure Functions Core Tools](https://learn.microsoft.com/azure/azure-functions/functions-run-local)
- [VS Code Azure Functions 拡張](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azurefunctions)

## Step 1: MCP サーバーを作成する

まずは、Azure Functions のことは一切考えずに、通常の MCP サーバーを作成してみましょう。

### プロジェクトの初期化

新しいディレクトリを作成して、プロジェクトを初期化します：

```bash
mkdir simple-calculator-mcp
cd simple-calculator-mcp
```

### 仮想環境の作成

Python の仮想環境を作成します：

```bash
python -m venv .venv
# macOS/Linux
source .venv/bin/activate
# Windows
.venv\Scripts\activate
```

### 必要なパッケージの定義

まず、依存関係を管理するために `requirements.txt` ファイルを作成します：

```txt
mcp>=1.5.0
```

この requirements.txt を使ってパッケージをインストールします：

```bash
pip install -r requirements.txt
```

### MCP サーバーのコード作成

`calculator.py` というファイルを作成して、シンプルな計算 MCP サーバーを実装します：

```python
#!/usr/bin/env python3
import sys
from mcp.server.fastmcp import FastMCP

# MCPサーバーの初期化
mcp = FastMCP("calculator", stateless_http=True)

@mcp.tool()
def add(a: float, b: float) -> float:
    """2つの数を足し算します

    Args:
        a: 最初の数
        b: 2番目の数

    Returns:
        足し算の結果
    """
    return a + b

@mcp.tool()
def subtract(a: float, b: float) -> float:
    """2つの数を引き算します

    Args:
        a: 最初の数（被減数）
        b: 2番目の数（減数）

    Returns:
        引き算の結果
    """
    return a - b

@mcp.tool()
def multiply(a: float, b: float) -> float:
    """2つの数を掛け算します

    Args:
        a: 最初の数
        b: 2番目の数

    Returns:
        掛け算の結果
    """
    return a * b

@mcp.tool()
def divide(a: float, b: float) -> float:
    """2つの数を割り算します

    Args:
        a: 最初の数（被除数）
        b: 2番目の数（除数）

    Returns:
        割り算の結果

    Raises:
        ValueError: bが0の場合
    """
    if b == 0:
        raise ValueError("0で割ることはできません")
    return a / b

if __name__ == "__main__":
    try:
        print("Calculator MCP Server を開始しています...")
        # streamable-http を使うと Uvicorn 上で HTTP サーバーが立ち上がり
        # クライアントは http://localhost:8000/mcp に接続できます
        mcp.run(transport="streamable-http")
    except Exception as e:
        print(f"エラーが発生しました: {e}", file=sys.stderr)
        sys.exit(1)
```

### requirements.txt の確認

依存関係が正しく定義されていることを確認しましょう。先ほど作成した `requirements.txt` の内容：

```txt
mcp>=1.5.0
```

シンプルな計算機能だけなので、MCP ライブラリのみで十分です。

### ローカルでのテスト実行

作成した MCP サーバーがローカルで正常に動作するか確認してみましょう：

```bash
python calculator.py
```

正常に動作すれば、以下のような出力が表示されるはずです：

```
Calculator MCP Server を開始しています...
INFO:     Started server process [26668]
INFO:     Waiting for application startup.
StreamableHTTP session manager started
INFO:     Application startup complete.
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
```

この状態で、Streamable HTTP で `http://localhost:8000/mcp` に接続可能になります。GitHub Copilot の Agent Mode や MCP Inspector などのクライアントから `http://localhost:8000/mcp` を指定して動作確認してください。

## Step 2: Azure Functions でホストできるようにする（必要なファイルの追加とコード修正）

このステップでは、先ほど作成した `calculator.py` を**最小限の変更**でカスタムハンドラーとして Azure Functions にホストできるようにします。主な作業は次の通りです：

- Functions のホストがカスタムハンドラーを起動・プロキシできるようにするための設定ファイル（`host.json`、`mcp-handler/function.json`、`local.settings.json`）を追加
- サーバーコードを、Functions が指定するポート（`FUNCTIONS_CUSTOMHANDLER_PORT`）で待ち受けるように変更
- `requirements.txt` がルートにあることを確認

### 1) `host.json` を追加

プロジェクトルートに以下の `host.json` を作成します（`arguments` の部分は実行するスクリプト名に合わせてください）：

```json
{
  "version": "2.0",
  "extensions": {
    "http": {
      "routePrefix": ""
    }
  },
  "customHandler": {
    "description": {
      "defaultExecutablePath": "python",
      "workingDirectory": "",
      "arguments": ["calculator.py"]
    },
    "enableForwardingHttpRequest": true,
    "enableHttpProxyingRequest": true
  }
}
```

この設定により、Functions ホストがカスタムハンドラープロセス（今回の場合は `python calculator.py`）を起動し、HTTP リクエストをハンドラーにプロキシします。

### 2) `mcp-handler/function.json` を追加

`mcp-handler` フォルダを作成し、その中に以下の `function.json` を置きます：

```json
{
  "bindings": [
    {
      "authLevel": "function",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": ["get", "post", "put", "delete", "patch", "head", "options"],
      "route": "{*route}"
    },
    {
      "type": "http",
      "direction": "out",
      "name": "res"
    }
  ]
}
```

このファイルにより、Functions の HTTP エンドポイント（`/mcp` など）に届いたリクエストがカスタムハンドラーへ転送されます。`authLevel` が `function` のため、実際の本番では Function Key が必要になります。

### 3) `local.settings.json` を追加

ローカル開発用に `local.settings.json` を作成します（Secrets は含めないでください）：

```json
{
  "IsEncrypted": false,
  "Values": {
    "FUNCTIONS_WORKER_RUNTIME": "custom"
  }
}
```

### 4) `calculator.py` を少し修正して Functions のポートを受け取るようにする

Functions のカスタムハンドラー実行時、ホストは`FUNCTIONS_CUSTOMHANDLER_PORT`環境変数でハンドラーの待ち受けポートを伝えます。これを使うようにコードを修正します。

変更前（今回の Step1 の状態）:

```python
# ...
mcp = FastMCP("calculator", stateless_http=True)
# ...
mcp.run(transport="streamable-http")
```

変更後（`FUNCTIONS_CUSTOMHANDLER_PORT` を受け取る）:

```python
import os
# ...
# FUNCTIONS_CUSTOMHANDLER_PORT を使ってポートを設定（ローカルや Azure 上で正しいポートを使うため）
mcp_port = int(os.environ.get("FUNCTIONS_CUSTOMHANDLER_PORT", 8080))

mcp = FastMCP("calculator", stateless_http=True, port=mcp_port)
# ...
mcp.run(transport="streamable-http")
```

- `stateless_http=True` と `transport="streamable-http"` はそのままにします。
- デフォルト値は `8080` にしていますが、`func start` 実行時に Functions ホストが自動的に適切なポートを設定します。

### 5) ローカルで Functions を起動して動作確認

1. `func start` を実行して Functions ホストを起動します：

   ```bash
   func start
   ```

2. 起動が成功すると、Azure Functions ホストが `http://localhost:7071`（デフォルト）で待ち受けます。MCP エンドポイントは`http://0.0.0.0:7071/mcp`で 接続できます。

---

以上で、サーバーを Azure Functions のカスタムハンドラーとして動かすための最小構成が揃いました。ここまで実施して `func start` でホストが起動し、`http://localhost:7071/mcp` にアクセスできることを確認したら、次は実際に Azure にデプロイしてリモートで動作確認する Step 3 に進みます。

## Step 3: Azure にデプロイしてリモートで動作確認（Portal で作成 → Azure CLI でデプロイ）

このステップでは、Azure ポータルで Azure Function リソースを作成し、作成したアプリに対して Azure CLI を使ってコードをデプロイします。スクリーン ショットを交えて手順をまとめます。

### 1) Portal で Function App を作成（GUI）

Portal で手順に従って Function App を作成します。主要なポイント：

- サブスクリプションとリソースグループを選択（新規作成しても良い）
- Publish: **Code** を選択
- Runtime stack: **Python**、バージョン **3.12** を選択
- OS: **Linux** を選択
- Plan: **Elastic (Consumption)** または Flex Consumption に相当するプランを選択
- ストレージアカウントを指定（新規で作成されることが多い）
- Networking: テストを楽にするなら **Enable public access** を一時的に有効にする

[Screenshot placeholder: Portal — Create Function App page]

> 備考: Portal の UI は頻繁に更新されます。上記の設定項目が見つからない場合は、各入力フィールドの説明を参照してください。

---

### 2) App Settings（アプリ設定）を確認/追加

Function App の「Configuration」->「Application settings」で以下を追加または確認します：

- `PYTHONPATH` を次のように設定（Linux の場合）：

```
/home/site/wwwroot/.python_packages/lib/site-packages
```

[Screenshot placeholder: Portal — Configuration -> Application settings]

CLI で設定する場合（例）：

```bash
az functionapp config appsettings set --name <FUNCTION_APP_NAME> --resource-group <RESOURCE_GROUP> --settings PYTHONPATH='/home/site/wwwroot/.python_packages/lib/site-packages'
```

---

### 3) コードをパッケージして Azure CLI でデプロイ（zip デプロイ）

1. プロジェクトルートで ZIP にまとめます（例: `deploy.zip`）

```bash
zip -r deploy.zip .
```

[Screenshot placeholder: Terminal — zip created]

2. Azure CLI で ZIP をデプロイします：

```bash
az functionapp deployment source config-zip --resource-group <RESOURCE_GROUP> --name <FUNCTION_APP_NAME> --src deploy.zip
```

[Screenshot placeholder: Terminal — az functionapp deployment source config-zip output]

デプロイが完了したら、Portal の Function App ブレードでデプロイ履歴や起動ログを確認します。

---

### 4) Function App のキーを取得して MCP クライアントを設定

1. Portal の Function App -> **Functions** -> **App keys**（または「Function keys / Host keys」）から `default` キーをコピーします。

[Screenshot placeholder: Portal — Function App keys]

2. VS Code の `.vscode/mcp.json` に以下のように `remote-mcp-server` を設定します（`x-functions-key` にコピーしたキーを貼り付け）：

```jsonc
"remote-mcp-server": {
  "type": "http",
  "url": "https://{FUNCTION_APP_NAME}.azurewebsites.net/mcp",
  "headers": {
    "x-functions-key": "{DEFAULT_KEY}"
  }
}
```

[Screenshot placeholder: VS Code — updating .vscode/mcp.json]

---

### 5) リモートでの動作確認

- VS Code の MCP 機能や Copilot の Agent Mode で `remote-mcp-server` を Start し、簡単なツール呼び出し（例：`add(3,5)`）を試します。
- Portal の「Log stream」でサーバー側のログを確認し、Uvicorn / StreamableHTTP の起動ログやリクエスト受信ログを確認します。

[Screenshot placeholder: VS Code — Start remote server]
[Screenshot placeholder: Portal — Log stream showing handler startup and requests]

---

### 6) 簡単なトラブルシュート

- 404 / 401 が出る
  - `routePrefix` が空になっているか (`host.json` の `routePrefix: ""`) を確認
  - `x-functions-key` を正しく設定しているか（キーのコピー漏れがよくある）
- ハンドラーが起動しない
  - `FUNCTIONS_CUSTOMHANDLER_PORT` を `calculator.py` で読めているか
  - `requirements.txt` の依存が正しくインストールされているか（`pip install -r requirements.txt`）
  - `PYTHONPATH` が正しく設定されているか
- ログが出ない
  - Portal の「Log stream」を開いてエラーを確認。必要なら `az webapp log tail --name <FUNCTION_APP_NAME> --resource-group <RESOURCE_GROUP>` を試す（環境によって利用可否が異なります）。

---

### 7) 成功判定

- `https://{FUNCTION_APP_NAME}.azurewebsites.net/mcp` に対して MCP クライアント（Copilot Agent / MCP Inspector）が接続でき、`add` 等のツール呼び出しに正しい応答が返ってくれば成功です。

---

次は、必要であれば APIM を使った認証や運用面の改善（監視・証跡・スケール設定など）を扱う章を追加できます。希望があれば続けて執筆します。
