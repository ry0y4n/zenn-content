---
title: "プライベート エンドポイントで閉域化された Document Intelligence を作る手順"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure"]
published: false
---

# 概要

この記事は以下のコンテンツを含んでいます。

-   Document Intelligence の作成とプライベート エンドポイントと組み合わせた閉域化の手順
-   閉域化された Document Intelligence を Document Intelligence Studio から利用するための注意点

# Document Intelligence の作成

Azure Portal で Document Intelligence を作成していきます。

![alt text](/images/private-document-intelligence-tips/create-di-1.png)

### ネットワーク の設定

ここだけ要注意です。

まず「種類」としては「Selected networks, configure network security for your Azure AI Services resource. (選択したネットワークとプライベート エンドポイント)」を選択してください。

そして、ファイアウォールの設定で、以下の 2 つの IP アドレスを許可してください。

-   Document Intelligence Studio の IP アドレス
-   お使いのコンピュータの IP アドレス

:::message
お使いのコンピュータの IP アドレスを調べる方法は以下の通りです。

**Bash コマンド：**

```bash
curl ifconfig.me
```

**PowerShell コマンド：**

```PowerShell
Invoke-RestMethod -Uri "https://ifconfig.me"
```

:::

![alt text](/images/private-document-intelligence-tips/create-di-2.png)

現時点での状況は以下の通りです。

-   Document Intelligence は作成されました。
-   Document Intelligence に対して、2 つの IP アドレスを許可するファイアウォールの設定が完了しました。
-   Document Intelligence へのアクセスはインターネット（パブリック IP アドレス）を経由して通信されます。

これをプライベート エンドポイントを使って閉域化（プライベート IP アドレス経由で通信に）します。

# プライベート エンドポイントの設定

Document Intelligence のプライベート エンドポイントを設定していきます。

リソース ページのネットワーク ブレードからプライベート エンドポイントを追加します。

![alt text](/images/private-document-intelligence-tips/configure-pe-1.png)

![alt text](/images/private-document-intelligence-tips/configure-pe-2.png)

使用するネットワークとしては、Document Intelligence 作成時に自動で作成された仮想ネットワーク（とサブネット）を流用します（もちろんしなくてもいいです）。

![alt text](/images/private-document-intelligence-tips/configure-pe-3.png)

# パブリック IP アドレス経由のアクセスを無効化

現時点では、仮想ネットワークから、パブリック IP アドレス経由でもプライベート エンドポイント経由でも Document Intelligence にアクセスできます。

なので、パブリック IP アドレス経由のアクセスを無効化します。

ネットワーク ブレードの「Firewalls and virtual networks」欄から選択されている仮想ネットワークをホバーした状態で右クリックして「Remove」を選択し保存することで許可を取り消し、パブリック IP アドレス経由のアクセスを無効化します。

![alt text](/images/private-document-intelligence-tips/disable-vnet-access-using-public.png)

# Document Intelligence Studio での動作確認

最後に、Document Intelligence Studio から Document Intelligence にアクセスできることを確認します。

![alt text](/images//private-document-intelligence-tips/check-studio-work.png)

# まとめ

[TODO] まとめを書く
