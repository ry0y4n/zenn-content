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

![alt text](/images/private-document-intelligence-tips/create-di-1.png)

![alt text](/images/private-document-intelligence-tips/create-di-2.png)

```bash
curl ifconfig.me
```

```PowerShell
Invoke-RestMethod -Uri "https://ifconfig.me"
```

# プライベート エンドポイントの設定

![alt text](/images/private-document-intelligence-tips/configure-pe-1.png)

![alt text](/images/private-document-intelligence-tips/configure-pe-2.png)

![alt text](/images/private-document-intelligence-tips/configure-pe-3.png)

# パブリック IP アドレス経由のアクセスを無効化

![alt text](/images/private-document-intelligence-tips/disable-vnet-access-using-public.png)

# Document Intelligence Studio での動作確認

![alt text](/images//private-document-intelligence-tips/check-studio-work.png)
