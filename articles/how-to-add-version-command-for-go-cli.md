---
title: "Cobra × GoReleaser 製 CLI ツールにバージョン コマンドを実装する方法"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "cli", "tips"]
published: true
---

こんにちは、モモスケです。
2025 年から Go に入門しまして、[Cobra](https://github.com/spf13/cobra) と [GoReleaser](https://goreleaser.com/) を使って CLI ツールを開発・リリースしています。
最近は [Envlate](https://github.com/ry0y4n/envlate) と [azurenv](https://github.com/ry0y4n/azurenv) というツールを作成しましたので、良ければご覧ください。

https://zenn.dev/microsoft/articles/generate-env-template-cli

https://zenn.dev/microsoft/articles/sync-app-settings

# 0. 背景

Cobra と GoReleaser を使って CLI ツールを開発・リリースしていると、バージョン情報を表示するコマンドが欲しくなる場面があります。今回は、実際に作成した CLI ツールで **`version`** サブコマンドを実装した際の手順をご紹介します。

「Go や GoReleaser を使って CLI を作ってみたけど、どうやってバージョン情報を埋め込めばいいの？」という方の参考になれば幸いです。

# 1. Cobra でサブコマンドを追加する

まずは Cobra を使って作成した CLI に、`version` サブコマンドを追加します。もしまだ [cobra-cli](https://github.com/spf13/cobra-cli) コマンドをインストールしていない場合は、以下のようにインストールしてください。

```bash
go install github.com/spf13/cobra-cli@latest
```

そして、プロジェクトのディレクトリで以下を実行します。

```bash
cobra-cli add version
```

すると、`version.go` ファイルが生成されます。これを編集してバージョン情報を表示する処理を書き足します。以下は例です。

```go
/*
Copyright © 2025 NAME HERE <EMAIL ADDRESS>
*/
package cmd

import (
	"fmt"

	"github.com/spf13/cobra"
)

var (
	version = "dev"
)

// versionCmd represents the version command
var versionCmd = &cobra.Command{
	Use:   "version",
	Short: "Show the version of azurenv CLI",
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Printf("azurenv %s", version)
	},
}

func init() {
	rootCmd.AddCommand(versionCmd)
}

```

上記のように書くだけでは、変数 version は常に "dev" のままです。そこで、ビルド時にバージョン情報を埋め込む必要があります。

# 2. GoReleaser でバージョン情報を埋め込む

[GoReleaser](https://goreleaser.com/cookbooks/using-main.version/) では、デフォルトで以下の 3 つの `ldflags` が設定されます。

-   `main.version`：現在の Git タグ（`v` 接頭辞は削除）またはスナップショット名
-   `main.commit`：現在の Git コミット SHA
-   `main.date`：ビルド日時（RFC3339 形式）

しかし、これらのデフォルト設定は「`main` パッケージに定義された変数」に対して有効に動作するものです。今回のように、`cmd` ディレクトリ内（`azurenv/cmd` パッケージ）に変数がある場合は、以下のように `-X` フラグでパッケージ名を正しく指定してあげる必要があります。

`.goreleaser.yaml` にある `builds` セクションに `ldflags` を追加し、`-X azurenv/cmd.version={{.Version}}` を指定します。

```yaml
builds:
    - env:
          - CGO_ENABLED=0
      goos:
          - linux
          - windows
          - darwin
      ldflags:
          - -s -w -X azurenv/cmd.version={{.Version}}
```

この設定により、GoReleaser がビルドする際に `cmd.version` にバージョン文字列（例：`1.2.3`）を代入してくれます。これで、`version` サブコマンドを実行すると埋め込んだバージョン番号が表示されるようになります。

# ハマったポイント

Go のリンカに渡す `-X` フラグでは、**"Go が認識している完全なインポートパス"** を指定しなければなりません。
[サンプルドキュメント](https://goreleaser.com/customization/builds/go/)のビルド セクションを見ると以下のような指定例が書いてあります。

```yaml
builds:
    ldflags:
        - -s -w -X main.build={{.Version}}
```

しかし、たとえば今回のように `cmd` ディレクトリにある変数を指定したい場合に、

```yaml
ldflags:
    - -s -w -X cmd.version={{.Version}}
```

としてしまうと、Go は「トップレベルに `cmd` という名前のパッケージがあるはずだが見つからない…」という状況になってしまいます。
実際には、`go.mod` が

```go.mod
module azurenv

go 1.23.5
```

となっているため、`cmd` ディレクトリは `azurenv/cmd` というパッケージパスとして扱われます。したがって正しくは、

```yaml
ldflags:
    - -s -w -X azurenv/cmd.version={{.Version}}
```

のように指定する必要がありました。

私と同じ Go 入門者の方は、この点に注意して設定してみてください。

# まとめ

-   **Cobra** で `version` サブコマンドを追加し、バージョン変数を宣言する。
-   **GoReleaser** でビルドする際に `-X パッケージパス.変数名=値` でバージョン情報を埋め込む。
-   `cmd.version` のようにサブパッケージに変数を置く場合は、**モジュール名を含めた完全なパッケージパス** で指定することに注意。

GoReleaser と Cobra の組み合わせは、Go で CLI ツールを開発するうえでとても便利な選択肢です。ただし、モジュール名やパッケージ構成をちゃんと把握していないと、リンカフラグの指定でハマってしまうことがあります。
この記事が、同様のケースに直面した際の参考になれば幸いです。ぜひ自分のプロジェクトでも試してみてください！

# 参考資料

-   [Using the main.version ldflag - GoReleaser](https://goreleaser.com/cookbooks/using-main.version/)
-   [Go - GoReleaser](https://goreleaser.com/customization/builds/go/)
-   [Cobra Generator](https://github.com/spf13/cobra-cli)
