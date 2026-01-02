---
title: ""
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# TL;DR

VS Code の [Diagram.net 拡張機能](https://marketplace.visualstudio.com/items?itemName=hediet.vscode-drawio) でカスタム アイコン ライブラリを使うための「SVG 変換」と「自動更新機能付き事前作成ライブラリ」の 2 機能を提供する「[SVG to Draw.io Library Converter](https://github.com/ry0y4n/diagramnet-icon-libraries)」を開発して公開しました。

# 背景

会社の年末オールハンズにて、社内のシニア エンジニアから、VS Code の Diagram.net 拡張機能でアーキテクチャ図を描く方法を教えてもらいました。

GitHub Copilot に作らせるファイル（`.drawio`）がかなり精度が良く、簡単なアーキテクチャ図であれば、ワンショットで作成できることがわかりました。

Diagram.net では、私がよく使う Azure をはじめとしたクラウド サービスのアイコン ライブラリが標準で用意されていますが、古いデザインだったり、新しいサービスのアイコンがなかったりしました。

![VS Code のスクリーンショット：右側に GitHub Copilot チャット欄（AI エージェント アプリのアーキテクチャ図を `.drawio` ファイルで作成させている）、左側に Diagram.net 拡張機能による埋め込みエディター（GitHub Copilot によって作成されたファイルを開いているが、いくつかのアイコンんが見つからず仮アイコンになっている。）](/images/vscode-diagramnet-custom-icon-library/vscode-diagram.net-extension-icon-problem.png)

普段、使っている Azure の最新アイコンをどうにか使いたいと思い、カスタム アイコンを追加する方法を調べたところ、「[最新の Azure アイコンと Visual Studio Code Draw.io Integration の活用](https://qiita.com/yaegashi/items/e661507765c48db09e2b)」という記事を見つけました。

この記事では以下のことが分かりました。

- VS Code の `settings.json` で、カスタム アイコン ライブラリ ファイル（`.xml`）の URL を指定することで、Diagram.net 拡張機能でカスタム アイコンを利用できる
- 有志が作成した Azure アイコン ライブラリ ファイルが GitHub で公開されている（[Azure Icon Libraries for Diagrams.net (Draw.io)](https://github.com/pacodelacruz/diagrams-net-azure-libraries)）
- 上記ライブラリは 3 年前に更新が止まっている

:::message
`settings.json` 例

```json
{
  "hediet.vscode-drawio.customLibraries": [
    {
      "entryId": "Azure Architecture Icons - Ai And Machine Learning",
      "libName": "Azure - Ai And Machine Learning",
      "url": "https://raw.githubusercontent.com/ry0y4n/diagramnet-icon-libraries/main/output/azure/ai-and-machine-learning.xml"
    },
    {
      // 他のライブラリも同様に追加可能
    }
  ]
}
```

:::

なので、自分で **自動更新機能付き** で最新の Azure アイコン ライブラリを作成し、さらにサービスのアイコンも含めたカスタム アイコン ライブラリを作成して公開しようと考えました。

# 原案

当初のアイデアでは、カスタム アイコン ライブラリを公開するプラットフォームや導線をどうするか、あまり考えていませんでした。

例えば、みんなが自由にライブラリを登録・共有できるような仕組みを作ることも考えましたし、CLI ツールを作って、各自が好きなアイコン ライブラリを作成できるようにすることも考えました（CLI は既存ツールが見つかりました：[最新の Azure アイコンを svg2mxlibrary で変換して diagrams.net (draw.io) で使用する](https://qiita.com/yaegashi/items/c0882e76baab72b13e39)）。

色々考えましたが、重要なコンポーネントは以下だと考えました。

1. 各サービスのアイコンを取得する Fetcher の実装
2. ライブラリが最新であるための自動更新機能

まずは MVP として、1. の Fetcher を実装し、GitHub Actions で 2. の自動更新機能を実装することにしました。そして生成した XML ファイルを GitHub 上でホスティングすることにしました。

# 実装してみて発覚したこと & ピボット

GitHub Copilot がある今、Fetcher の実装は思ったより簡単でした。各アイコンは各サイトで Zip 形式で配布されており、それをダウンロードして展開し、SVG ファイルを抽出して、Diagram.net 用の XML ファイル（mxGraph XML）に変換するロジックを実装しました。

ただ、アイコンを拡張機能で使ってみようとしたときにあることに気づきました。

**「最新のアイコンが無い！！！」**

なんと公式提供のアイコンでも、最新のアイコンが提供されていないことが分かりました。例えば、11 月下旬にリリースされた Microsoft Foundry のアイコンは、1 月 1 日現在まだ提供されていません。

このように、公式提供のアイコンが最新でない場合、Fetcher を実装して自動更新機能をつけても、解決できない課題があることが分かりました。

そこで、本ツールに **「SVG 変換機能」** を追加することにしました。これにより、ユーザーが自分で用意した SVG アイコンを Diagram.net 用の XML ファイルに変換できるようになります。

# 最終的な機能

最終的に以下の 2 機能を提供するツールとして実装しました。

1. **SVG 変換機能**: ユーザーが用意した SVG アイコンを Diagram.net 用の XML ファイルに変換する機能
2. **自動更新機能付き事前生成ライブラリ**: Azure アイコン ライブラリをはじめとした複数のカスタム アイコン ライブラリを自動更新機能付きで提供する機能

そして、両機能を GitHub Pages としてまとめて公開しました。

[Diagrams.net Icon Libraries](https://ry0y4n.github.io/diagramnet-icon-libraries/)

### SVG 変換機能

![SVG ファイルをアップロードして、Diagram.net 用の XML ファイルに変換・ダウンロードして、VS Code でライブラリを設定してカスタム アイコンが使えることを確認するデモ](/images/vscode-diagramnet-custom-icon-library/converter-demo.gif)

### 自動更新機能付き事前生成ライブラリ

![事前生成のライブラリ群を使うためのスニペットをポータルからコピーして VS Code に貼り付けて設定するデモ](/images/vscode-diagramnet-custom-icon-library/prebuilt-custom-icon-demo.gif)

# 今後の展望

今後は、以下のような展開を考えています。

- 他のクラウド サービス（AWS、Google Cloud など）のアイコンライブラリの追加
- ユーザーが自分でライブラリを追加・共有できる仕組みの構築
- ユーザーからのフィードバックを元にした機能改善
