---
title: "既存の MCP サーバーを最小限の変更で Azure Functions にホストしてみよう"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure", "azurefunctions", "mcp", "python"]
published: true
publication_name: "microsoft"
---

# はじめに

Model Context Protocol（MCP）は、Anthropic が開発した AI エージェントとツール間の標準化されたプロト（以下略...!!）

これまでも Azure Functions で MCP サーバーを開発することは可能でしたが（[サンプル コード](https://github.com/Azure-Samples/remote-mcp-functions-python)）、それは Azure Functions のライブラリを使って**最初から Azure Functions での動作を前提として**開発する必要がありました。

しかし、多くの開発者は既にローカル環境で動作する MCP サーバーを構築済みで、「このサーバーをクラウドで動かしたいけど、一から書き直すのは面倒だな...」と感じているのではないでしょうか？

そんな要望に応えるために、**既存の MCP サーバーを最小限の変更で Azure Functions にホストできる**公式サンプルが Microsoft からリリースされました！

https://github.com/Azure-Samples/mcp-sdk-functions-hosting-python

このサンプルの最大の魅力は、**Azure Functions を全く意識せずに開発した既存の MCP サーバーを、ほんの少しの設定ファイル追加だけでクラウド（Azure）にホストできる**ことです。

この記事では、実際にこのサンプルを使って以下の流れを体験していきます：

1. **MCP サーバーの作成** - Azure を意識せずにシンプルな MCP サーバーを作る
2. **Azure Functions 対応** - 設定ファイルを追加するだけで既存の MCP サーバーを Azure Functions 対応に
3. **ローカル実行** - `func start`でローカルで Azure Functions 環境を再現
4. **デプロイ** - 実際に Azure にデプロイしてリモートで動作確認

# 手順

今回は学習目的で、シンプルな **四則演算ツール** を提供する MCP サーバーを作成します。

## 前提条件

この記事を進めるために、以下が必要です：

### 環境

-   Python 3.12（推奨）
-   Azure アカウント（無料アカウントで OK）

### ツール

-   [Azure CLI](https://learn.microsoft.com/ja-jp/cli/azure/install-azure-cli?view=azure-cli-latest)
-   [Azure Functions Core Tools](https://learn.microsoft.com/azure/azure-functions/functions-run-local)

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

この状態で、Streamable HTTP 方式で **`http://localhost:8000/mcp`** に接続可能になります。**GitHub Copilot の Agent Mode** や **MCP Inspector** などのクライアントから **`http://localhost:8000/mcp`** を指定して動作確認してください。

## Step 2: Azure Functions でホストできるようにする（必要なファイルの追加とコード微修正）

このステップでは、先ほど作成した `calculator.py` を**最小限の変更**でカスタム ハンドラーとして Azure Functions にホストできるようにします。主な作業は次の通りです：

-   Functions のホストがカスタム ハンドラーを起動・プロキシできるようにするための設定ファイル（`host.json`、`mcp-handler/function.json`、`local.settings.json`）を追加
-   サーバーコードを、Functions が指定するポート（`FUNCTIONS_CUSTOMHANDLER_PORT`）で待ち受けるように変更
-   `requirements.txt` がルートにあることを確認

### 1) `host.json` を追加

プロジェクト ルートに以下の `host.json` を作成します（`arguments` の部分は実行するスクリプト名に合わせてください）：

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

この設定により、Functions ホストがカスタム ハンドラー プロセス（今回の場合は `python calculator.py`）を起動し、HTTP リクエストをハンドラーにプロキシします。

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
            "methods": [
                "get",
                "post",
                "put",
                "delete",
                "patch",
                "head",
                "options"
            ],
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

このファイルにより、Functions の HTTP エンドポイント（`/mcp` など）に届いたリクエストがカスタム ハンドラーへ転送されます。`authLevel` が `function` のため、実際の本番では Function Key が必要になります。

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

Functions のカスタム ハンドラー実行時、ホストは`FUNCTIONS_CUSTOMHANDLER_PORT`環境変数でハンドラーの待ち受けポートを伝えます。これを使うようにコードを修正します。

変更前（今回の Step1 の状態）:

```python
# ...
mcp = FastMCP("calculator", stateless_http=True)
```

変更後（`FUNCTIONS_CUSTOMHANDLER_PORT` を受け取る）:

```python
import os
# ...
# FUNCTIONS_CUSTOMHANDLER_PORT を使ってポートを設定（ローカルや Azure 上で正しいポートを使うため）
mcp_port = int(os.environ.get("FUNCTIONS_CUSTOMHANDLER_PORT", 8080))
mcp = FastMCP("calculator", stateless_http=True, port=mcp_port)
```

-   ポート番号の指定をデフォルト値は `8080` にしていますが、`func start` 実行時に Azure Functions ホストが自動的に適切なポートを設定します。

### 5) ローカルで Functions を起動して動作確認

1. Azure Functions Core Tools を使って Azure Functions ホストを起動します：

    ```bash
    func start
    ```

2. 起動が成功すると、Azure Functions ホストが **`http://localhost:7071`**（デフォルト）で待ち受けます。MCP エンドポイントは **`http://0.0.0.0:7071/mcp`** で接続できます。

改めて、GitHub Copilot や MCP Inspector 等の MCP クライアントから **`http://localhost:7071/mcp`** に接続して動作確認してください。

## Step 3: Azure にデプロイしてリモートで動作確認（Azure ポータルで作成 → Azure CLI でデプロイ）

このステップでは、Azure ポータルで Azure Functions リソースを作成し、作成したアプリに対して Azure CLI を使ってコードをデプロイします。スクリーン ショットを交えて手順をまとめます。

### 1) Azure Functions リソースを作成（GUI）

ランタイム スタックは **Python 3.12** にしてください。その他の設定はデフォルトのままで問題ありません。

![Azure ポータルのリソース作成画面](/images/host-existing-mcp-server-on-azure-functions/create-function.png)

### 2) App Settings（アプリ設定）を確認/追加

Azure Functions の「環境変数」で以下を追加します：

| 名前       | 値                                                    |
| ---------- | ----------------------------------------------------- |
| PYTHONPATH | /home/site/wwwroot/.python_packages/lib/site-packages |

![環境変数の設定](/images/host-existing-mcp-server-on-azure-functions/update-app-config.png)

CLI で設定する場合：

```bash
az functionapp config appsettings set --name <FUNCTION_APP_NAME> --resource-group <RESOURCE_GROUP> --settings PYTHONPATH='/home/site/wwwroot/.python_packages/lib/site-packages'
```

### 3) コードをデプロイ

Azure Functions Core Tools でデプロイします：

```bash
func azure functionapp publish <FUNCTION_APP_NAME> --python
```

デプロイが完了したら、Portal の Azure Functions ブレードでデプロイ履歴や起動ログを確認します。

### 4) Azure Functions のキーを取得して MCP クライアントを設定

1. Azure ポータルの Azure Functions -> **アプリ キー** から `default` キーをコピーします。

![アプリ キーの確認](/images/host-existing-mcp-server-on-azure-functions/copy-app-key.png)

このアプリ キーを MCP クライアントのヘッダー `x-functions-key` に設定することで、認証付きで Azure Functions のエンドポイントにアクセスできます。

1. VS Code であれば `.vscode/mcp.json` に以下のように設定します（`x-functions-key` にコピーしたキーを貼り付け）：

```json
{
    "servers": {
        "Remote-Calculator-MCP": {
            "type": "http",
            "url": "https://{FUNCTION_APP_NAME}.azurewebsites.net/mcp",
            "headers": {
                "x-functions-key": "{DEFAULT_KEY}"
            }
        }
    }
}
```

### 5) リモートでの動作確認

VS Code の MCP 機能や Copilot の Agent Mode で `Remote-Calculator-MCP` を Start し、簡単なツール呼び出しを試します。

![リモート MCP サーバーの動作確認](/images/host-existing-mcp-server-on-azure-functions/remote-mcp-work.png)

# まとめ

最小変更（設定追加＋ポート対応）で既存の MCP サーバーを Azure Functions に載せ、ローカル検証からデプロイまで一気通貫で確認できました。

皆さんも、ぜひこの方法で既存の MCP サーバーを Azure Functions にホストしてみてください！
