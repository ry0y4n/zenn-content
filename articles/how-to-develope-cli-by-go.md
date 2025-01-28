---
title: "【完全ガイド】Go で CLI ツールを作り、リリースまで一気通貫！"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "cli", "githubactions", "cicd"]
published: true
---

こんにちは、モモスケです！
本記事では、Go 言語を使って簡単な CLI ツールを作成し、リリースまでの流れを丁寧に解説します。

具体的には、以下の 3 ステップをハンズオン形式で進めます。

1. **Cobra を使った CLI ツールの作成**
2. **GoReleaser を使ったリリース**
3. **GitHub Actions を使った自動化パイプライン構築**

この記事を読み終える頃には、自分だけの CLI ツールを簡単に開発・公開できるようになっているはずです！

:::message
**今回のハンズオンの完成形は、以下のリポジトリで確認できます。**
https://github.com/ry0y4n/my-cli-tool
:::

# 1. 環境構築

まずは開発環境を整えます。

## 1.1 Go のインストール

以下の公式ドキュメントを参考に、お使いの OS に合わせて **Go** をインストールしてください。

https://go.dev/doc/install

以下のコマンドでバージョンが表示されればインストール成功です。

```bash
go version
```

## 1.2 Cobra のインストール

[Cobra](https://github.com/spf13/cobra) は CLI アプリケーションを簡単に作成するためのフレームワークです。以下のコマンドでインストールします。

```bash
go get -u github.com/spf13/cobra@latest
```

## 1.3 Cobra CLI のインストール

Cobra CLI は Cobra の CLI ツールです。以下のコマンドでインストールします。

```bash
go install github.com/spf13/cobra-cli@latest
```

## 1.4 GoReleaser のインストール

以下の公式ドキュメントを参考に、GoReleaser をインストールしてください。

https://goreleaser.com/install/

おすすめは `go install` です（OS に寄らずインストールできます）。

```bash
go install github.com/goreleaser/goreleaser/v2@latest
```

以下のコマンドでバージョンが表示されればインストール成功です。

```bash
goreleaser --version
```

# 2. Cobra で簡単な CLI ツールを作ってみよう

## 2.1 プロジェクトの作成

まずは新しいディレクトリを作成し、Go モジュールを初期化します。

```bash
mkdir my-cli-tool
cd my-cli-tool
go mod init my-cli-tool
```

`go.mod` が作成されます。

```plaintext
module my-cli-tool

go 1.23.5
```

## 2.2 Cobra で初期化

Cobra を使ってプロジェクトのテンプレートを生成します。

```bash
cobra-cli init
```

以下のような構成のファイルが作成されます。

```bash
my-cli-tool/
├── cmd/
│   └── root.go
├── go.mod
├── go.sum
└── main.go
```

`cmd/root.go` がコマンドのエントリーポイントです。

## 2.3 「Hello, World!」を実装

次に、`cmd/root.go` の `Run` 関数に処理を追加します。

```go
Run: func(cmd *cobra.Command, args []string) {
    fmt.Println("Hello, World!")
}
```

:::details cmd/root.go の全体コード

```go
/*
Copyright © 2025 NAME HERE <EMAIL ADDRESS>
*/
package cmd

import (
	"fmt"
	"os"

	"github.com/spf13/cobra"
)

// rootCmd represents the base command when called without any subcommands
var rootCmd = &cobra.Command{
	Use:   "my-cli-tool",
	Short: "A brief description of your application",
	Long: `A longer description that spans multiple lines and likely contains
examples and usage of using your application. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.`,
	// Uncomment the following line if your bare application
	// has an action associated with it:
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("Hello, World!")
	},
}

// Execute adds all child commands to the root command and sets flags appropriately.
// This is called by main.main(). It only needs to happen once to the rootCmd.
func Execute() {
	err := rootCmd.Execute()
	if err != nil {
		os.Exit(1)
	}
}

func init() {
	// Here you will define your flags and configuration settings.
	// Cobra supports persistent flags, which, if defined here,
	// will be global for your application.

	// rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $HOME/.my-cli-tool.yaml)")

	// Cobra also supports local flags, which will only run
	// when this action is called directly.
	rootCmd.Flags().BoolP("toggle", "t", false, "Help message for toggle")
}

```

:::

以下のコマンドで実行してみましょう。

```bash
go run main.go

or

# macOS / Linux
go build -o my-cli-tool
my-cli-tool
# Windows
go build -o my-cli-tool.exe
my-cli-tool.exe
```

出力結果は以下のようになります。

```bash
Hello, World!
```

これで **`Hello, World!`** と出力する CLI ツールの完成です！

## 2.4 新しいサブコマンドの作成

以下のコマンドで、新しいサブコマンドを追加できます。

```bash
cobra-cli add greet
```

これにより、`cmd/greet.go` ファイルが生成されます。このファイルを編集して、挨拶メッセージを表示するコマンドを作成します。

```go
Run: func(cmd *cobra.Command, args []string) {
    fmt.Println("Hello, CLI user!")
}
```

:::details cmd/greet.go の全体コード

```go
/*
Copyright © 2025 NAME HERE <EMAIL ADDRESS>
*/
package cmd

import (
	"fmt"

	"github.com/spf13/cobra"
)

// greetCmd represents the greet command
var greetCmd = &cobra.Command{
	Use:   "greet",
	Short: "A brief description of your command",
	Long: `A longer description that spans multiple lines and likely contains examples
and usage of using your command. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.`,
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("Hello, CLI user!")
	},
}

func init() {
	rootCmd.AddCommand(greetCmd)

	// Here you will define your flags and configuration settings.

	// Cobra supports Persistent Flags which will work for this command
	// and all subcommands, e.g.:
	// greetCmd.PersistentFlags().String("foo", "", "A help for foo")

	// Cobra supports local flags which will only run when this command
	// is called directly, e.g.:
	// greetCmd.Flags().BoolP("toggle", "t", false, "Help message for toggle")
}

```

:::

追加したコマンドを実行してみます。

```bash
go run main.go greet

or

# macOS / Linux
go build -o my-cli-tool
my-cli-tool greet
# Windows
go build -o my-cli-tool.exe
my-cli-tool.exe greet
```

出力結果は以下のようになります。

```bash
Hello, CLI user!
```

:::message
サブコマンドの用途としては以下のようなものがあります。

-   `version`：バージョン情報の表示
-   `init`：初期化処理
-   複数機能の切り替え（`my-cli-tool process-A` or `my-cli-tool process-B`）

:::

# 3. GoReleaser を使った CLI ツールのリリース

CLI ツールをリリースするには、[GoReleaser](https://goreleaser.com/) を活用します。

## 3.1 設定ファイルの作成

以下のコマンドで、`.goreleaser.yaml` を生成します。

```bash
goreleaser init
```

GoRelear ではこの設定ファイルをもとに、CLI を各プラットフォーム向けにビルドし、リリースします。

## 3.2 ローカルでリリーステスト

以下のコマンドでローカル環境でビルドを試してみましょう。

```bash
goreleaser release --snapshot --clean
```

-   `--snapshot` はリリースのシミュレーションを行うためのオプションです。
-   `--clean` は前回のビルド結果を削除するオプションです。

生成されたバイナリは **`dist/`** ディレクトリに保存されます。

例えば、Windows 向けのバイナリは `.\dist\my-cli-tool_windows_amd64_v1\my-cli-tool.exe` に生成されます。各自の OS に合わせて実行してみましょう。

```bash
# Windows (x86_64) の場合
.\dist\my-cli-tool_windows_amd64_v1\my-cli-tool.exe
# Hello, World!
```

これで、ローカルでのリリースが完了しました。
あとは、GitHub にリリース用のパイプラインを構築することで、自動リリースが可能です。

# 4. GitHub Actions で自動リリース

GitHub Actions を使えば、タグを push するだけで自動的にリリースを行えます。

## 4.1 ワークフローの作成

以下の内容で `.github/workflows/release.yml` を作成します。

```yaml
name: release

on:
    push:
        tags:
            - "v*"

jobs:
    release:
        runs-on: ubuntu-latest
        permissions:
            contents: write
        steps:
            - name: Checkout
              uses: actions/checkout@v4
              with:
                  fetch-depth: 0
            - name: Set up Go
              uses: actions/setup-go@v5
            - name: Run GoReleaser
              uses: goreleaser/goreleaser-action@v6
              with:
                  distribution: goreleaser
                  version: "~> v2"
                  args: release --clean
              env:
                  GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

このワークフローは、`v*` というタグが push されたときにリリースを実行します。

GoReleaser の公式アクションである [goreleaser-action](https://github.com/goreleaser/goreleaser-action) を使うことで、GitHub Actions で GoReleaser を実行できます。

## 4.2 リリースのトリガー

リリースタグを作成して push します。

```bash
# 現時点の変更を commit して push
git add .
git commit -m "IMPL: CLI says Hello, World!"
git push

# タグを作成して push
git tag v1.0.0
git push origin v1.0.0
```

これで、GitHub Actions がリリースを実行します。

![GitHub Actions が実行している様子](/images/how-to-develop-cli-by-go/github-action-run.png)

リリースが成功すると、GitHub のリリースページにて各プラットフォーム向けのバイナリがダウンロードできるようになります。

![リリース ページの様子](/images/how-to-develop-cli-by-go/release-page.png)

**これで、Go で CLI ツールを開発し、リリースするためのパイプラインが完成しました。**

あとは、ユーザーがバイナリがダウンロードして使用可能な場所に配置したり、`PATH` を通すことで使うことができます。

# (Optional) 5. インストール スクリプトの作成

ユーザーが CLI ツールを簡単にインストールできるように、インストール スクリプトを用意しておき、`README.md` に記載しておくと便利です。

## macOS / Linux

以下のスクリプトを `install.sh` として保存します。

```bash
#!/bin/bash

# If no version is specified as a command line argument, fetch the latest version.
if [ -z "$1" ]; then
    VERSION=$(curl -s https://api.github.com/repos/<GITHUB_USER_NAME>/<GITHUB_REPO_NAME>/releases/latest | grep -o '"tag_name": *"[^"]*"' | sed 's/"tag_name": *"//' | sed 's/"//')
    if [ -z "$VERSION" ]; then
        echo "Failed to fetch the latest version"
        exit 1
    fi
else
    VERSION=$1
fi

OS=$(uname -s)
ARCH=$(uname -m)
URL="https://github.com/<GITHUB_USER_NAME>/<GITHUB_REPO_NAME>/releases/download/${VERSION}/<GITHUB_REPO_NAME>_${OS}_${ARCH}.tar.gz"

echo "Start to install."
echo "VERSION=$VERSION, OS=$OS, ARCH=$ARCH"
echo "URL=$URL"

TMP_DIR=$(mktemp -d)
curl -L $URL -o $TMP_DIR/<GITHUB_REPO_NAME>.tar.gz
tar -xzvf $TMP_DIR/<GITHUB_REPO_NAME>.tar.gz -C $TMP_DIR
sudo mv $TMP_DIR/<GITHUB_REPO_NAME> /usr/local/bin/<GITHUB_REPO_NAME>
sudo chmod +x /usr/local/bin/<GITHUB_REPO_NAME>

rm -rf $TMP_DIR

if [ -f "/usr/local/bin/<GITHUB_REPO_NAME>" ]; then
  echo "[SUCCESS] <GITHUB_REPO_NAME> $VERSION has been installed to /usr/local/bin"
else
  echo "[FAIL] <GITHUB_REPO_NAME> $VERSION is not installed."
fi
```

このスクリプトを実行すると、最新バージョンの CLI ツールがダウンロードされ、`/usr/local/bin` にインストールされます。

```bash
curl -fsSL https://raw.githubusercontent.com/<GITHUB_USER_NAME>/<GITHUB_REPO_NAME>/main/install.sh | sh
```

## Windows

以下のスクリプトを `install.ps1` として保存します。

```powershell
param(
    [Parameter(Mandatory=$false)]
    [ValidateSet("x86_64", "arm64", "i386")]
    [string]$Arch = "x86_64"
)

$ErrorActionPreference = 'Stop'

[string]$Version

if (-not $Version) {
    Write-Host "Fetching the latest version..."
    try {
        $response = Invoke-WebRequest -Uri "https://api.github.com/repos/<GITHUB_USER_NAME>/<GITHUB_REPO_NAME>/releases/latest" -UseBasicParsing
        $json = $response.Content | ConvertFrom-Json
        $Version = $json.tag_name
        if (-not $Version) {
            Write-Error "Failed to fetch the latest version"
            exit 1
        }
    } catch {
        Write-Error "Failed to fetch the latest version"
        exit 1
    }
}

$url = "https://github.com/<GITHUB_USER_NAME>/<GITHUB_REPO_NAME>/releases/download/$Version/<GITHUB_REPO_NAME>_Windows_$Arch.zip"
$output = "$env:temp\<GITHUB_REPO_NAME>.zip"
$installDir = "$env:LocalAppData\<GITHUB_REPO_NAME>"

Write-Host "Downloading <GITHUB_REPO_NAME> version $Version for architecture: $Arch"
Write-Host "Download URL: $url"

Invoke-WebRequest -Uri $url -OutFile $output

Write-Host "Extracting <GITHUB_REPO_NAME>"
Expand-Archive -Path $output -DestinationPath $installDir -Force

Write-Host "Adding <GITHUB_REPO_NAME> to PATH"
$oldPath = [Environment]::GetEnvironmentVariable('Path', [System.EnvironmentVariableTarget]::User)
$newPath = "$oldPath;$installDir"
[Environment]::SetEnvironmentVariable('Path', $newPath, [System.EnvironmentVariableTarget]::User)

Write-Host "Installation complete. Please restart your terminal."
```

このスクリプトを実行すると、最新バージョンの CLI ツールがダウンロードされ、`C:\Users\<USER_NAME>\AppData\Local` にインストールされ、また `PATH` に追加されます。

### x86_64 (default)

```powershell
powershell -Command "Invoke-WebRequest -Uri https://raw.githubusercontent.com/<GITHUB_USER_NAME>/<GITHUB_REPO_NAME>/main/install.ps1 -OutFile install.ps1; .\install.ps1"
```

### arm64

```powershell
powershell -Command "Invoke-WebRequest -Uri https://raw.githubusercontent.com/<GITHUB_USER_NAME>/<GITHUB_REPO_NAME>/main/install.ps1 -OutFile install.ps1; .\install.ps1 -Arch arm64"
```

### i386

```powershell
powershell -Command "Invoke-WebRequest -Uri https://raw.githubusercontent.com/<GITHUB_USER_NAME>/<GITHUB_REPO_NAME>/main/install.ps1 -OutFile install.ps1; .\install.ps1 -Arch i386"
```

以上で、インストール スクリプトの作成が完了しました。

# 6. まとめ

この記事では、Go で CLI ツールを開発し、リリースする手順を解説しました。

-   **Cobra で CLI を作成**
-   **GoReleaser でバイナリをビルド**
-   **GitHub Actions で自動化**

シンプルなツールから始めて、ぜひ独自の CLI ツールを作成してみてください！

# 参考資料

-   [GoReleaser 公式ドキュメント](https://goreleaser.com/)
-   [goreleaser-action - GitHub](https://github.com/goreleaser/goreleaser-action)
-   [Cobra - GitHub](https://github.com/spf13/cobra)
-   [Cobra CLI - GitHub](https://github.com/spf13/cobra-cli)
-   [Go で CLI コマンドを自作して公開するまでの道のり](https://zenn.dev/tttol/articles/c7dfc74d27e45d)
-   [GoReleaser で Go 製 CLI のリリースを自動化＆ Homebrew でインストールできるようにする](https://zenn.dev/kou_pg_0131/articles/goreleaser-usage)
-   [Go で Cobra を使って CLI を作成してみた](https://qiita.com/zumax/items/7c45a01abb31f0494823)
