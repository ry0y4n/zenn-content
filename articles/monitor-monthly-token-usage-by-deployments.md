---
title: "【Azure OpenAI】デプロイ毎のトークン使用量/月を監視したい"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [azure, microsoft]
published: false
---

# はじめに

Azure OpenAI リソースのトークン使用量について「デプロイメント毎」にまた「月毎」に監視したい。

# 背景

Azure OpenAI リソースのトークン使用量を監視する場合、アラート ルールを作って監視したいと思うのが自然です。

しかし、、、

### メトリック アラート ルールで監視しようとすると

トークン使用量のメトリック（[TokenTransaction](https://learn.microsoft.com/en-us/azure/ai-services/openai/monitor-openai-reference#:~:text=ModelDeploymentName%20and%20ModelName.-,TokenTransaction,-Count)）でメトリック アラート ルールを作ろうとすると、ルックバック期間の最大が「24 時間」までしかなく、月次の監視ができません。

![alt text](/images/monitor-monthly-token-usage-by-deployments/metric-max-lookback.png)

### ログ検索アラート ルールで監視しようとすると

ログ検索アラート ルールなら、Kusto クエリを使って色々柔軟にできそうです。
まずは、Log Analytics ワークスペースにメトリック データを取り込む必要があるので、診断設定を作成します。

![alt text](/images/monitor-monthly-token-usage-by-deployments/diagnostic-setting.png)

`AzureMetrics` テーブルにメトリック データが流れてくるのですが、デプロイメントに関する情報は取得できません。

![alt text](/images/monitor-monthly-token-usage-by-deployments/no-deployment-info.png)

さらに言えば、ログ検索アラート ルールの粒度の最大が「2 日」までしか無いので、月次の監視ができません。

![alt text](/images/monitor-monthly-token-usage-by-deployments/log-alert-granularity-limit.png)

（↑ のクエリは月のトークン使用量を算出できるものとなっているが、最大粒度（2 日）でラップされてしまうので、アラート ルールは想定の動作をしない。という罠の話はまた別の記事で詳しく書きたい。）

# 解決策

このような（アラート ルールでは柔軟性が足りない）場合、Azure Functions または Logic Apps などを使って、自前で監視する方法が考えられます。

Logic Apps は Low-Code で便利なのですが、せっかくなので今回は Azure Functions (TypeScript) で監視を実装してみます。

### やること

1. メトリック データの取得
2. デプロイメント毎のトークン使用量の算出
3. 値に応じてアラートを送信

### 1．メトリック データの取得

<!-- - P1M は無理だった -->
<!-- - 監視閲覧者がいる -->
