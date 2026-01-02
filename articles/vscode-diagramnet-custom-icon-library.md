---
title: "【VS Code の diagrams.net でカスタム アイコンを使う】SVG→ライブラリ変換＆自動更新ライブラリを公開"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode", "diagramsnet", "drawio", "azure", "microsoft"]
published: true
---

# TL;DR

VS Code の diagrams.net 拡張機能（[Draw.io Integration](https://marketplace.visualstudio.com/items?itemName=hediet.vscode-drawio)）で **カスタム アイコン ライブラリ（`.xml`）** を扱いやすくするために、次の 2 つをまとめたポータル「[SVG to diagrams.net Library Converter](https://ry0y4n.github.io/vscode-diagrams-icon-builder/)」を公開しました。

- **SVG 変換**: 手元の SVG を diagrams.net 用ライブラリ（mxGraph XML / `.xml`）へ変換
- **自動更新つき事前生成ライブラリ**: Azure などのアイコンライブラリを GitHub Actions で定期生成し、URL で配布

![ヒーロー イメージ：ポータル サイトのスクリーンショット](/images/vscode-diagramnet-custom-icon-library/converter.png)

- ポータル: https://ry0y4n.github.io/vscode-diagrams-icon-builder/
- GitHub リポジトリ: https://github.com/ry0y4n/vscode-diagrams-icon-builder

# できること

- VS Code 上の埋め込みエディターで、最新のクラウド アイコンや自作アイコンを **ライブラリとして追加**できる
- `settings.json` に貼り付けるだけのスニペットで、**事前生成ライブラリ**をすぐ使える
- 公式配布が追いついていないアイコンも、**自前 SVG で補完**できる

# 使い方

## 1. 事前生成ライブラリを使う（おすすめ）

![事前生成のライブラリ群を使うためのスニペットをポータルからコピーして VS Code に貼り付けて設定するデモ](https://github.com/ry0y4n/vscode-diagrams-icon-builder/blob/main/docs/images/prebuilt-custom-icon-demo.gif?raw=true)

1. [ポータル](https://ry0y4n.github.io/vscode-diagrams-icon-builder/) を開く
2. 使いたいライブラリを選び、VS Code 用のスニペットをコピーする
3. VS Code の `settings.json`（ユーザー設定、またはワークスペースの `.vscode/settings.json`）へ貼り付ける
4. VS Code を再読み込みして、diagrams.net のライブラリ一覧に追加されていることを確認する

:::message
`settings.json` の例（URL で `.xml` を参照）

```json
{
  "hediet.vscode-drawio.customLibraries": [
    {
      "entryId": "Azure Architecture Icons - Ai And Machine Learning",
      "libName": "Azure - Ai And Machine Learning",
      "url": "https://raw.githubusercontent.com/ry0y4n/vscode-diagrams-icon-builder/main/output/azure/ai-and-machine-learning.xml"
    }
  ]
}
```

- `entryId`: 内部識別子（重複しない値にしておくと安全です）
- `libName`: UI に表示されるライブラリ名
- `url`: ライブラリ XML（mxGraph XML）への URL

:::

## 2. 手元の SVG をライブラリ化する（不足分を自前で補完）

![SVG ファイルをアップロードして、diagrams.net 用の XML ファイルに変換・ダウンロードして、VS Code でライブラリを設定してカスタム アイコンが使えることを確認するデモ](https://github.com/ry0y4n/vscode-diagrams-icon-builder/blob/main/docs/images/converter-demo.gif?raw=true)

1. ポータルの **Custom SVG Converter** へ SVG ファイル（複数可）をアップロードする
2. 生成された `.xml` をダウンロードする
3. `.xml` を任意のフォルダに配置する。
4. そのファイル パスを `hediet.vscode-drawio.customLibraries` に追加する

```json
{
  "hediet.vscode-drawio.customLibraries": [
    {
      "entryId": "azure:ai-and-machine-learning",
      "libName": "Azure Architecture Icons - Ai And Machine Learning",
      "file": "\\\\wsl.localhost\\Ubuntu-24.04/home/rhyakuta/codes/vscode-digramnet-custom-icon/output/azure/ai-and-machine-learning.xml"
    }
  ]
}
```

> 注意: `url` の代わりに `file` プロパティを使い、パスを指定することができます。WSL2 on Windows の場合は UNC パス形式（`\\wsl.localhost\...`）で指定する必要があります。これは拡張機能側の仕様です。
>
> :::message
> 例えば WSL2 上のパス（`/home/rhyakuta/codes/vscode-digramnet-custom-icon/output/azure/ai-and-machine-learning.xml`）を指定した場合、拡張機能側で使っている VS Code ライブラリのファイル処理（`Uri.file()`）で Windows 側のパス（`C:\home\rhyakuta\codes\vscode-digramnet-custom-icon\output\azure\ai-and-machine-learning.xm`）に変換されてしまい、ファイルが見つからなくなります。VS Code は Windows 上で動作しているためです。
>
> 僕はこれでかなりハマりました...!!!
> :::

# 開発した背景

会社の年末オールハンズで、社内のシニア エンジニアから「GitHub Copilot + VS Code の diagrams.net 拡張でアーキテクチャ図を描く」方法を教えてもらいました。

GitHub Copilot に `.drawio` ファイルを書かせると精度が想像以上に良く、簡単な図ならワンショットで作れることが分かりました。

一方で、diagrams.net の標準ライブラリ（Azure など）は「古いデザインのまま」「新しいサービスのアイコンがまだ無い」といったケースがあり、図の見た目や正確性に影響していました。

![VS Code のスクリーンショット：右側に GitHub Copilot チャット欄（AI エージェント アプリのアーキテクチャ図を `.drawio` ファイルで作成させている）、左側に diagrams.net 拡張機能による埋め込みエディター（生成されたファイルを開いているが、いくつかのアイコンが見つからず仮アイコンになっている）](/images/vscode-diagramnet-custom-icon-library/vscode-diagram.net-extension-icon-problem.png)

普段、使っている Azure の最新アイコンをどうにか使いたいと思い、カスタム アイコンを追加する方法を調べたところ、「[最新の Azure アイコンと Visual Studio Code Draw.io Integration の活用](https://qiita.com/yaegashi/items/e661507765c48db09e2b)」という記事を見つけました。

この記事では以下のことが分かりました。

- VS Code の `settings.json` で、カスタム アイコン ライブラリ（`.xml`）の URL を指定すると、diagrams.net 拡張でそのライブラリを利用できる
- 有志が作成した Azure アイコン ライブラリ ファイルが GitHub で公開されている（[Azure Icon Libraries for Diagrams.net (Draw.io)](https://github.com/pacodelacruz/diagrams-net-azure-libraries)）
- 上記ライブラリは 3 年前に更新が止まっている

「最新で使える状態を保つ」こと自体が面倒になりがちなので、**自動更新つき**で配布できる形にして、手間を最小化しようと考えました。

# 原案

当初のアイデアでは、カスタム アイコン ライブラリを公開するプラットフォームや導線をどうするか、あまり考えていませんでした。

例えば、みんなが自由にライブラリを登録・共有できるような仕組みを作ることも考えましたし、CLI ツールを作って、各自が好きなアイコン ライブラリを作成できるようにすることも考えました（CLI は既存ツールが見つかりました：[最新の Azure アイコンを svg2mxlibrary で変換して diagrams.net (draw.io) で使用する](https://qiita.com/yaegashi/items/c0882e76baab72b13e39)）。

色々考えましたが、重要なコンポーネントは以下だと考えました。

1. **各サービスのアイコンを取得する Fetcher の実装**
2. **ライブラリが最新であるための自動更新機能**

まずは MVP として 1 の Fetcher を実装し、GitHub Actions のスケジュール実行で 2 の自動更新を回す方針にしました。生成した XML は GitHub でホスティングします。

# 実装してみて発覚したこと & ピボット

GitHub Copilot がある今、Fetcher の実装自体は思ったより簡単でした。各ベンダーが Zip で配布しているアイコンをダウンロードして展開し、SVG を抽出して diagrams.net 用の XML（mxGraph XML）に変換する、という流れです。

ただ、アイコンを拡張機能で使ってみようとしたときにあることに気づきました。

**「最新のアイコンが無い！！！」**

なんと「公式配布のアイコン」でさえ、新サービスの追加直後などは更新が追いつかない場合がありました。

このように、公式提供のアイコンが最新でない場合、Fetcher を実装して自動更新機能をつけても、解決できない課題があることが分かりました。

そこで本ツールに **SVG 変換機能**を追加し、ユーザーが手元の SVG で不足分を補えるようにしました。

# 最終的な機能（まとめ）

最終的に以下の 2 機能を提供するツールとして実装しました。

1. **SVG 変換機能**: ユーザーが用意した SVG アイコンを diagrams.net 用の XML ファイルに変換する機能
2. **自動更新機能付き事前生成ライブラリ**: Azure アイコン ライブラリをはじめとした複数のカスタム アイコン ライブラリを自動更新機能付きで提供する機能

そして、両機能を GitHub Pages としてまとめて公開しました。

# 仕組み（自動更新の概要）

事前生成ライブラリは、GitHub Actions のスケジュール実行で「取得 → 変換 → 出力」を繰り返し、生成物（`.xml`）を配布しています。VS Code 側は `settings.json` に URL を登録するだけなので、更新のたびに手動で入れ替える必要がありません。

```yaml
name: Update Icon Libraries

on:
  # Run weekly on Sunday at 00:00 UTC
  schedule:
    - cron: "0 0 * * 0"

  # Allow manual trigger
  workflow_dispatch:

  # Run on push to main (for testing)
  push:
    branches:
      - main
    paths:
      - "src/**"
      - "docs/**"
      - ".github/workflows/**"

jobs:
  generate:
    runs-on: ubuntu-latest

    permissions:
      contents: write
      pages: write
      id-token: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Generate icon libraries
        run: python -m src.main --output output --clean

      - name: Check for changes
        id: check_changes
        run: |
          git diff --quiet output/ || echo "changed=true" >> $GITHUB_OUTPUT

      - name: Commit and push changes
        if: steps.check_changes.outputs.changed == 'true'
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add output/
          git commit -m "chore: Update icon libraries [skip ci]"
          git push

      - name: Setup Pages
        uses: actions/configure-pages@v4

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: "docs"

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

# 今後の展望

今後は、以下のような展開を考えています。

- 他のクラウド サービス（AWS、Google Cloud など）のアイコンライブラリ追加
- ユーザーが自分でライブラリを追加・共有できる仕組みの構築
- ユーザーからのフィードバックを元にした機能改善

> 正直、普段使わないサービスの Fetcher を実装するモチベはあまりないので、もしコミュニティからの貢献があれば嬉しいです...!!!

もし「このアイコンセットも欲しい」「ここが分かりにくい」などあれば、リポジトリの Issue / PR で気軽に教えてください。
