---
title: "ログ検索アラート ルールのためのカスタム ログ挿入方法"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [azure]
published: false
---

# 概要

ログ検索アラート ルールの動作を確認したり、テストする時どうしていますか？アプリを作ってログを出力して確認している方もいるかもしれません。しかし、その方法だとあまり効率的ではありません。

そこでこの記事では、Logic Apps を使って任意のログを任意のタイミングで出力する方法を紹介します。これにより、ログ検索アラート ルールの動作を確認する際に、より効率的にテストができるようになります。

# 登場人物

-   Logic Apps
-   Log Analytics ワークスペース
-   データ収集ルール
-   データ収集エンドポイント

# 0. Log Analytics ワークスペース作成

もし無ければ、ログを収集したい Log Analytics ワークスペースを作成しておく

# 1. Logic Apps

## リソース作成

特に必要な設定はありません。空の Logic Apps を作成してください（今回、ホスティング プランは「従量課金」で作成しました）。

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/logic-apps-overview.png)

## アプリ設定

# 2. データ収集エンドポイント作成

# 3. データ収集ルール作成

# 4. ログ出力

# (Optional) 5. ログ検索アラート ルールのテスト

# まとめ

# 参考資料
