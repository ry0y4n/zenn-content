---
title: "【知らなくて良いけど知ってると嬉しい】Log Analytics ワークスペースのアクセス モード"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [azure, azuremonitor, monitor, loganalytics]
published: true
publication_name: "microsoft"
---

モモスケです。[Microsoft Azure Tech Advent Calendar 2024](https://qiita.com/advent-calendar/2024/microsoft-azure-tech) の 4 日目の記事となります。

# 概要

この記事は以下の内容を含んでいます。

-   Log Analytics ワークスペースのアクセス モードについて
-   知らず知らずのうちにアクセス モードを享受している瞬間
-   アクセス モードを知らないと困ること

# まず問題です！

以下の画像を見て、どこかおかしい場所が無いか考えてみてください。

![alt text](/images/log-analytics-access-mode/first-quiz.png)

与えられた情報以外のこと（閉域化や権限）は考えず、どこかおかしい場所を見つけてください。

### ヒント

この状態では、検知したいログが Log Analytics ワークスペースに送信されていても、アラートは発報しません。

### どこがおかしいか分かりましたか？

:::details 正解はこちら
正解は「**アラート ルールのスコープ**」です。
:::

:::message alert

### ミニ解説

スコープを Log Analytics ワークスペース以外（リソースやリソース グループ）に設定すると、「**リソース コンテキスト**」というアクセス モードが適用されます。今回の場合、指定したリソース グループ（`RG-A`）およびその内部リソースのリソース ID が関連付けられているログ データのみが検索可能となります。

監視対象の VM は別のリソース グループ (`RG-B`) に存在しているため検索対象外となり、ログが検知されません。
:::

# Log Analytics ワークスペースのアクセス モードとは

https://learn.microsoft.com/ja-jp/azure/azure-monitor/logs/manage-access?tabs=portal#access-mode

アクセス モードはアラート ルールを含め Log Analytics ワークスペース内のログ データにアクセスする際の権限を制御する機能です。

## ワークスペース コンテキスト

ユーザーがアクセス許可を持つワークスペース内のすべてのログを表示できます。このモードのクエリの範囲は、**ワークスペース内で**アクセス権を持つすべてのデータです。

## リソース コンテキスト

特定のリソース、リソース グループ、またはサブスクリプションに関してワークスペースにアクセスする場合に使用されます。このモードのクエリの範囲は、**そのリソースに関連するデータを含むすべてのワークスペース**です。

## 「そのリソースに関連するデータ」とは？

どのようにして「そのリソースに関連するデータ」を定義するのかというと、`_ResourceId` フィールドを使用してリソース ID を取得し、そのリソース ID がリソース コンテキストに関連付けられているログ データのみが検索可能となります。

つまり、`_ResourceId` フィールドに指定したリソースのリソース ID が格納されているログ データすべてが検索対象となります。

![alt text](/images/log-analytics-access-mode/resource-id-log.png)
例として、VM (`test-vm`) に関するログの `_ResourceId` フィールドには、VM のリソース ID が格納されています。

## 【復習】先ほどのクイズで何が起こったのか？

### リソース コンテキストのアクセス モードが適用された

スコープに `RG-A` を選択したことで、リソース コンテキスト モードでのログ検索となりました。その結果、`RG-A` およびその内部のリソースに関連付けられたログ データのみが検索可能となり、`RG-B` に存在する VM のログは検索対象外となりました。

もし、`RG-A` に VM が存在していた場合は、ログが検知されます。

# これって実はアクセス モードを感じています

## ワークスペース コンテキストを感じている瞬間

-   Log Analytics ワークスペースのリソース ページからログを検索する時は、**ワークスペース コンテキスト**でログを検索しています。

以下は、先ほどの VM (`test-vm`) に関するログを、保存先とは別の Log Analytics ワークスペースから検索している様子です。

![alt text](/images/log-analytics-access-mode/workspace-context-sample.png)

この場合、ワークスペース コンテキスト モードでログを検索していますので、検索範囲はこのワークスペース内のすべてのデータとなります。よってログは検知されません。

## リソース コンテキストを感じている瞬間

-   監視対象のリソース ページからログを検索する時は、**リソース コンテキスト**でログを検索しています。

以下は、VM (`test-vm`) に関するログを、VM 自身のリソース ページから検索している様子です。

![alt text](/images/log-analytics-access-mode/resource-context-sample.png)

この場合、リソース コンテキスト モードでログを検索していますので、検索範囲は Log Analytics ワークスペースに依らず、そのリソースに関連付けられたデータとなります。よってログは検知されます。

# クイズの改善案

### 案 1：スコープを格納先の Log Analytics ワークスペースに変更する

ワークスペース コンテキスト モードでログを検索するため、ログが検知されるようになります。

![alt text](/images/log-analytics-access-mode/quiz-refac-plan-1.png)

### 案 2：スコープを監視対象の VM に変更する

リソース コンテキスト モードでログを検索するため、ログが検知されるようになります。

![alt text](/images/log-analytics-access-mode/quiz-refac-plan-2.png)

## どちらがいいのか？

### 基本的にはワークスペース コンテキスト モード

理由として、特定の Log Analytics ワークスペースをスコープとして設定することで、ワークスペース内のすべてのログ データにアクセスできます。この状況はクエリの検索範囲がより直観的で分かりやすいです。

### リソース コンテキスト モードの利用例

監視対象リソースのログが複数の Log Analytics ワークスペースに分散している場合、リソース コンテキスト モードを使用することで、全範囲からそのリソースに関連付けられたデータを検索することができます。

詳細は以下のリンクを参照してください。

https://learn.microsoft.com/ja-jp/azure/azure-monitor/logs/cross-workspace-query

# まとめ

-   Log Analytics ワークスペースのアクセス モードには「ワークスペース コンテキスト」と「リソース コンテキスト」があります。
-   普段使っている Azure Portal でのログ検索も、アクセス モードによって異なる検索範囲が適用されています。
-   アラート ルールのスコープを設定する際は、アクセス モードを意識して設定しましょう。

# 参考

-   [アクセス モード - Log Analytics ワークスペースへのアクセスを管理する](https://learn.microsoft.com/ja-jp/azure/azure-monitor/logs/manage-access?tabs=portal#access-mode)
-   [Azure Monitor 内の Log Analytics ワークスペース、アプリケーション、リソース全体のデータにクエリを実行する](https://learn.microsoft.com/ja-jp/azure/azure-monitor/logs/cross-workspace-query)
-   [Log Analytics ワークスペースのアクセス制御モードの違いについて](https://jpazmon-integ.github.io/blog/LogAnalytics/AccessControlMode/)
