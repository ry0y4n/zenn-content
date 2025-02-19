---
title: "カスタムインストラクションの一歩先！GitHub Copilot の指示を分割管理しよう"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
publication_name: "microsoft"
---

GitHub Copilot は、日々のコーディング作業を効率化する強力なツールです。もともと、**Custom Instructions** を用いて、プロジェクト固有のライブラリやコーディングスタイルに合わせたコード生成を実現してきました。

:::message
この記事では、カスタム インストラクション自体の詳説は省略します。カスタム インストラクションに入門するには土田さんの「**[GitHub Copilot にカスタムインストラクションで最新技術や独自ルールを教え込む](https://zenn.dev/microsoft/articles/github-copilot-custom-instructions)**」をご参照ください。
:::

しかし、従来の方法では **単一の Markdown ファイルしかインプットできない** ため、複数の指示を場面ごとに分割して管理することは難しいという課題がありました。

そこで登場したのが「**Reusable Prompt Files**」です。これにより、役割ごとに異なる指示ファイルを用意し、必要なものだけを取り出してコンテキストとして反映させることが可能になりました。

# Custom Instructions の基本

まずは、従来の GitHub Copilot のカスタムインストラクションについておさらいします。  
**Custom Instructions** を使うと、例えば以下のようなコード生成時のルールを設定できます。

```json
"github.copilot.chat.codeGeneration.instructions": [
  {
    "text": "Always add a comment: 'Generated by Copilot'."
  },
  {
    "text": "In TypeScript always use underscore for private field names."
  },
  {
    "file": "code-style.md" // import instructions from file `code-style.md`
  }
]
```

上記の例では、直接テキストを指定する方法と、Markdown ファイルからインポートする方法の両方が利用されています。さらに、`.github/copilot-instructions.md` に記述することで、カスタム インストラクションを利用できる点も魅力です。

# Reusable Prompt Files の魅力と概要

Reusable Prompt Files では、従来のように単一ファイルに全ての指示をまとめる必要がなく、複数の Markdown ファイルを用途に応じて分割できます。 具体的には、以下のような手順で利用します。

1. **任意の場所にコード規則やルール ファイル（Markdown）を作成**  
   例：`code-style.md` や `code-style2.md` といったファイルに、プロジェクト固有のコーディングルールを記述します。

2. **.github/prompts/.prompt.md ファイルを作成**  
   このファイル内で、必要な Markdown ファイルを下記のいずれかの記法でインポートします。

    ```
    [index](../../code-style.md)
    #file:../../code-style.md
    ```

3. **VS Code の Chat, Edit, Inline Chat 欄からプロンプトを呼び出す**  
   クリップアイコンをクリックし「Prompt...」を選択、`.github/prompts` を指定することで、そのチャットセッションのコンテキストに対してプロンプトファイルが適用されます。

# プロンプトファイルの具体例

以下は、実際のプロンプトファイルの一例です。

## .prompt.md の内容

```
[index](../../code-style.md)
[index](../../code-style2.md)
```

## code-style.md の内容

````
Always add code comments. The template is as follows:
```python
# [COPILOT COMMENT] Display a message to the console
print("Hello, World!")
```
````

## code-style2.md の内容

````

Always print a message with color to the console. The template is as follows:

```python
print("\033[31mThis is red color\033[0m")
```
````

この設定により、チャットに入力すると、.prompt.md とそれが参照する Markdown ファイル（この例では code-style.md と code-style2.md）の両方の規則が反映され、**コメントの挿入**と**出力時の色付け**の両方が適用されたコード生成が実現されます。

# なぜ Reusable Prompt Files が有用なのか

-   **柔軟な管理**
    プロンプトを用途別にファイル分割することで、プロジェクトごとや場面ごとに最適な指示をピックアップできるため、より精度の高いコード生成が可能に。

-   **再利用性の向上**
    一度作成したプロンプトファイルは、複数のプロジェクトで使い回すことができ、設定の一貫性が保たれます。

-   **開発体験の向上**
    チャットやインライン編集中に適用されるプロンプトが、自動的に複数の規則を組み合わせた形で反映されるため、開発者はよりシームレスな体験を享受できます。

# まとめ

GitHub Copilot の Reusable Prompt Files 機能は、従来のカスタムインストラクションの柔軟性を一段と高める革新的なツールです。

-   **設定方法の簡単さ**：Markdown 形式で規則を分割し、必要な時にインポートするだけで OK。
-   **多様なシナリオへの対応**：コメントの自動挿入や、出力のフォーマット指定など、用途に応じた複数のルールを同時に利用可能。

これにより、開発現場におけるコード生成のクオリティが向上し、プロジェクトごとのカスタマイズも容易になります。ぜひ、あなたのプロジェクトでもこの新機能を活用して、よりスマートな開発ライフを実現してみてください!

# 参考所法

-   [Reusable prompt files (experimental)](https://code.visualstudio.com/docs/copilot/copilot-customization#_reusable-prompt-files-experimental)
