---
title: "GitHub Copilot Chat in Windows Terminal を試してみよう"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [githubcopilot, windowsterminal, aoai, openai]
published: false
publication_name: "microsoft"
---

10 月 29 日、Windows Terminal (カナリー版) で試験中のターミナル チャットに GitHub Copilot が追加されました。Windows Terminal で GitHub Copilot とチャットすることができます。

# GitHub Copilot Chat in Windows Terminal とは

-   Widnows Terminal (カナリー版) の ターミナル チャット機能を使って、GitHub Copilot とチャットすることができる機能です。

## ターミナル チャットとは？

-   Windows Terminal (カナリー版) で、任意の AI サービスと統合できる機能です。
-   AI サービス プロバイダーは、GitHub Copilot、Azure OpenAI、OpenAI があります。

https://learn.microsoft.com/ja-jp/windows/terminal/terminal-chat

# セットアップ

## 必要なもの

-   GitHub Copilot へのアクセス権
-   カナリー版の Windows Terminal

:::message
組織で GitHub Copilot へのアクセスを管理している場合、管理者が「Copilot in the CLI」設定を無効にしている場合、Windows Terminal で Copilot を使用することができないの要注意です。

設定については [organization 内での Copilot のポリシーの管理](https://docs.github.com/ja/copilot/managing-copilot/managing-github-copilot-in-your-organization/managing-policies-for-copilot-in-your-organization) をご参考ください。
:::

## Windows Terminal (カナリー版) のインストール

以下のリンクから Windows Terminal (カナリー版) をインストールしてください。

https://github.com/microsoft/terminal#installing-windows-terminal-canary

![alt text](/images/github-copilot-chat-in-windows-terminal/install-canary.png)

## ターミナル チャットの設定

設定のタブから、ターミナル チャットの設定を開き、GitHub Copilot の契約がアクティブな GitHub アカウントでログインします。

![alt text](/images/github-copilot-chat-in-windows-terminal/terminal-chat-setting.png)

![alt text](/images/github-copilot-chat-in-windows-terminal/github-auth.png)

![alt text](/images/github-copilot-chat-in-windows-terminal/set-as-active-porovider.png)

# 試してみる

Terminal Chat を開き、GitHub Copilot に質問をしてみます。

![alt text](/images/github-copilot-chat-in-windows-terminal/open-terminal-chat.png)

![alt text](/images/github-copilot-chat-in-windows-terminal/terminal-chat-ui.png)

再生ボタンのアイコンをクリックすると、自動的にターミナルに貼り付けられるようになっています。

![alt text](/images/github-copilot-chat-in-windows-terminal/result.png)

![alt text](/images/github-copilot-chat-in-windows-terminal/reflect-terminal.png)

複雑なシェルスクリプトやコマンドなどは貼り付けるのが面倒なこともあるので自動的にコピペしてくれるのは便利です。シェルスクリプトやコマンドのヘルプを求める際は、ここで質問すると良いかもしれません。

# 制限

一方で、残念なポイントが以下です。

![alt text](/images/github-copilot-chat-in-windows-terminal/limitation.png)

他にもいろいろな質問をしてみましたが、基本的にはターミナルのコマンドに関する質問に対してのみ回答してくれるようです。

僕はてっきり、コードやプロジェクトに関する質問に対しても回答してくれると思っていたのですが、残念ながらコードに関する質問には回答してくれませんでした。

OSS を Clone してきたときにとりあえず、VSCode で解説してもらって超速理解するという運用をよくしているのですが、Windows Terminal で完結できるようになれば、もっと魅力的になるかもしれません。

# 他 AI サービス プロバイダー

どちらもシークレット情報を入力するだけで良いので、簡単に設定できます。

### Azure OpenAI

エンドポイントとシークレット キーを入力して、Azure OpenAI と連携することができます。

![alt text](/images/github-copilot-chat-in-windows-terminal/use-aoai.png)

### OpenAI

シークレット キーを入力して、OpenAI と連携することができます。

![alt text](/images/github-copilot-chat-in-windows-terminal/use-openai.png)

# まとめ

-   Windows Terminal (カナリー版) で GitHub Copilot とチャットすることができる機能を試してみました。
-   GitHub Copilot の他に、Azure OpenAI、OpenAI とも連携することができます。
-   現状、ターミナルのコマンドに関する質問に対してのみ回答してくれるようです。

# 参考

-   [GitHub Copilot in Windows Terminal](https://devblogs.microsoft.com/commandline/github-copilot-in-windows-terminal/)
-   [Windows ターミナルで GitHub Copilot に質問をする](https://docs.github.com/ja/copilot/using-github-copilot/asking-github-copilot-questions-in-windows-terminal)
-   [ターミナル チャット (試験段階)](https://learn.microsoft.com/ja-jp/windows/terminal/terminal-chat)
