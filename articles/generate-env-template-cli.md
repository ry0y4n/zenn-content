---
title: "【Envlate】.env ファイルから .env.template ファイルを生成するクロスプラットフォーム CLI ツール"
emoji: "🔖"
type: "tech"
topics: ["go", "cli", "env"]
published: true
publication_name: "microsoft"
---

こんにちは、モモスケです。  
今回は、`.env` ファイルから簡単に `.env.template` ファイルを生成できる CLI ツール **Envlate** を開発しましたのでご紹介します。

## Envlate とは？

Envlate は Go 言語で開発された **クロスプラットフォームな CLI ツール** です。Go でビルドしているため、**Windows / macOS / Linux** を含むさまざまな環境で動作します。また、Envlate は以下のような特徴を持ちます。

-   `.env` や `.env.local` など、任意の環境変数ファイルからテンプレートファイル（`.env.template` や `.env.local.template`）を生成
-   インストールスクリプトを一度実行するだけで使えるため、依存ライブラリ不要
-   ファイル パスを指定して生成した場合は、元のファイルと同じディレクトリにテンプレートファイルを作成

リポジトリはこちらに公開しています。

https://github.com/ry0y4n/envlate

### デモ

![](/images/generate-env-template-cli/envlate-demo.gif)

---

## 背景

プロジェクトをチーム開発していると、ソースコード管理 (Git など) に `.env` ファイルを含めないようにするケースが多いです。しかし、新しくプロジェクトに参加したメンバーが環境変数の設定を行うために、何をどう設定すればよいかを伝える方法に悩むことがあります。

そんなときに便利なのが `.env.template` ファイルです。これは、`.env` ファイルと同じ構造を持ちながら **値を空やコメントにする** などして、どのような環境変数が存在するかのみを共有できるようにしたものです。

### 例

`.env`: コンフィデンシャルな情報を含むため、`.gitignore` に追加して、リモート リポジトリにはアップロードしない運用が一般的。

```
SAMPLE_ENDPOINT="https://api.example.com"
SAMPLE_KEY="1234567890abc"
SAMPLE_TOKEN="qwertyuiopasdfghjklzxcvbnm"
SAMPLE_DB_USER="admin"
SAMPLE_DB_PASS="admin"
```

`.env.template`: `.env` と同じ構造を持ちながら、値を空やコメントにして、どのような環境変数が存在するかのみを共有。

```
SAMPLE_ENDPOINT=
SAMPLE_KEY=
SAMPLE_TOKEN=
SAMPLE_DB_USER=
SAMPLE_DB_PASS=
```

### 既存の類似ツール

.env ファイルと .env.template 間の生成や同期をサポートするツールとしては、下記のようなものが既に存在しています。

-   [gen-env-template](https://github.com/mrsauravsahu/gen-env-template)
-   [env-template](https://github.com/hitgo00/env-template)

どちらも npm パッケージとして提供されているため、Node.js がインストールされている環境なら手軽に利用できます。一方で、Node.js を入れていない環境（Docker イメージ内や CI/CD の実行環境など）では手間がかかる場合もあるかと思います。

そこで今回は **Go 言語で実装し、ビルド済みのバイナリを配布** することで、なるべく多くの環境で手軽に実行できるようにしたツール **Envlate** を開発しました。

---

## Envlate の主な機能

-   **テンプレート生成**  
    既存の `.env` などのファイルを元に、テキストの値部分だけ空やコメントなどに置き換えたテンプレートファイル (`.env.template`) を生成します。
-   **シンプルなコマンド**  
    オプションなしで `envlate` と入力するだけで、カレントディレクトリの `.env` を見つけてそのテンプレートを生成してくれます。
-   **柔軟なファイル指定**  
    デフォルトではカレントディレクトリの `.env` を探索しますが、`--file` オプションで任意のパスを指定することも可能です。指定ファイルと同じディレクトリにテンプレートファイルを作成します。

---

## インストール方法

### macOS / Linux

```bash
curl -fsSL https://raw.githubusercontent.com/ry0y4n/envlate/main/install.sh | sh
```

### Windows

#### x86_64 (default)

```powershell
powershell -Command "Invoke-WebRequest -Uri https://raw.githubusercontent.com/ry0y4n/envlate/main/install.ps1 -OutFile install.ps1; .\install.ps1"
```

#### arm64

```powershell
powershell -Command "Invoke-WebRequest -Uri https://raw.githubusercontent.com/ry0y4n/envlate/main/install.ps1 -OutFile install.ps1; .\install.ps1 -Arch arm64"
```

#### i386

```powershell
powershell -Command "Invoke-WebRequest -Uri https://raw.githubusercontent.com/ry0y4n/envlate/main/install.ps1 -OutFile install.ps1; .\install.ps1 -Arch i386"
```

インストール スクリプトが完了すれば、そのまま envlate コマンドが使えるようになります。

## 使い方

```bash
# カレント ディレクトリの .env から生成
envlate
```

カレントディレクトリに `.env` があれば、`.env.template` が作成されます。

### 特定ファイルを指定したい場合

```bash
envlate --file path/to/envfile
```

`path/to/envfile` のパスにあるファイル（例：`./src/.env.production`）を元に、同じディレクトリにテンプレートファイルを生成します（`./src/.env.production.template`）。

## まとめ

Envlate を使えば、環境変数のテンプレートファイルを手軽に管理・共有 できます。Go 言語でビルドしているため、Node.js や他のランタイムをインストールする手間がないのも大きな魅力です。

ぜひ皆さんのコンピュータに `Envlate` をインストールしてみてください。

当方にとって初めての Go 開発で至らない点があるかと思います。気になる点や改善案があれば、GitHub で Issue や PR を立てていただけると嬉しいです！

# 参考資料

-   [Go で CLI コマンドを自作して公開するまでの道のり](https://zenn.dev/tttol/articles/c7dfc74d27e45d)

    -   インストール スクリプト作成で、記事内でご共有のコードをほとんど流用させていただきました！

-   [GoReleaser で Go 製 CLI のリリースを自動化＆ Homebrew でインストールできるようにする](https://zenn.dev/kou_pg_0131/articles/goreleaser-usage)

    -   GoReleaser 導入から GitHub Actions での自動リリースまで参考にしました！

-   [Go で Cobra を使って CLI を作成してみた](https://qiita.com/zumax/items/7c45a01abb31f0494823)
    -   Cobra を導入する際に参考にしました！

---

P.S. お気づきかと思いますが、「`Envlate`」は `env` と `template` のシャレです。
