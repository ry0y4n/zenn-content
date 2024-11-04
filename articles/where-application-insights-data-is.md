---
title: "データの保存先はどこ？ Application Insights のよくある誤解"
emoji: "🙅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [azure, microsoft, applicationinsight, loganalytics]
published: true
publication_name: "microsoft"
---

# 目的

業務やネット サーフィン中に、 Application Insights に関する「小さな誤解」を目にする機会が多々あります。些細な点ではありますが、誤解を解消することが役立つと感じたため、まとめてお伝えしたいと思います。

# よくある誤解

- 「Application Insights に保存する」
- 「データは Application Insights に送って監視しています」

これらの表現は厳密には正確ではありません。今回は、上記の表現がどのように誤解を招くのかを説明し、正確な表現についてもご理解いただければと思います。

# Application Insights とは

Application Insights は、 Azure Monitor の一部で、アプリケーションのパフォーマンスや利用状況を可視化・分析するために使用されるコンポーネントです。

![alt](/images/where-application-insights-data-is/app-insights-overview-blowout.png)

https://learn.microsoft.com/ja-jp/azure/azure-monitor/app/app-insights-overview#logic-model

# 実際のデータ送信先

テレメトリ（データ）の送信先・保存先は Application Insights ではなく、 **Log Analytics ワークスペース**です。  
 Application Insights はテレメトリを収集・送信するための役割を担い、データの保存自体は Log Analytics ワークスペースが担当しています。

そのため、 Application Insights 自体に**データを保存する機能はありません**。

# 誤解の原因について

私は以下の二点を原因として捉えています。

## 1. クラシック版 Application Insights とワークスペース版 Application Insights の混同

以前の「クラシック版」 Application Insights は、独自のテレメトリ保管場所を持っており、当時は「Application Insights にデータを送る」という表現が正しかったと言えます。この名残で「Application Insights に保存する」と誤解される方もいらっしゃるのかもしれません。

しかし、 2020 年 9 月に「ワークスペース版」として GA （一般公開）された現在の Application Insights では、テレメトリの保管機能が Log Analytics ワークスペースに委譲されました。また、クラシック版は 2024 年 2 月にリタイア（廃止）となっています。

クラシック版とワークスペース版の違いについては、[こちらの公式ブログ](https://jpazmon-integ.github.io/blog/applicationInsights/aboutDifferentTypesOfAi/)で詳しく説明されていますので、本記事では割愛いたします。

## 2. クラシック版の影響を受けたテーブル名の不一致

ワークスペース版 Application Insights でテレメトリを収集し、 Log Analytics ワークスペースにデータを保存する場合、以下のような現象が見られます。

- Azure Portal の Log Analytics ワークスペース リソース ページから「ログ」ブレードを選択し、 Kusto クエリを使用して Application Insights によって収集されたデータを確認できます。

  ```
  AppRequests // アプリへのリクエスト データ
  ```

- Azure Portal の Application Insights リソース ページからも「ログ」ブレードを選択し、同じく Kusto クエリでデータを確認できます。

  ```
  requests // アプリへのリクエスト データ（あれ、テーブル名が違う？）
  ```

- しかし、上記 2 つのブレードで同一のデータを参照する際、使用するテーブル名が異なります。

この違いによって、一部のユーザーは「別のデータが Application Insights に保存されているのでは？」と誤解されることがあります。実際には以下のとおり、Azure Portal 状の参照元リソースによって異なるテーブル名でデータが表示される仕様になっており、参照するデータは同一のものとなっています。

### テーブル名の違い（[参考](https://learn.microsoft.com/ja-jp/previous-versions/azure/azure-monitor/app/convert-classic-resource#table-structure)）

| レガシ テーブル名   | 新しいテーブル名       | 説明                                                                                 |
| ------------------- | ---------------------- | ------------------------------------------------------------------------------------ |
| availabilityResults | AppAvailabilityResults | 可用性テストの要約データ                                                             |
| browserTimings      | AppBrowserTimings      | クライアントのパフォーマンスに関するデータ                                           |
| dependencies        | AppDependencies        | TrackDependency() を介して記録されたアプリケーションから他コンポーネントへの呼び出し |
| customEvents        | AppEvents              | カスタム イベント                                                                    |
| customMetrics       | AppMetrics             | カスタム メトリック                                                                  |
| pageViews           | AppPageViews           | 各 Web サイトのビューに関するデータとブラウザー情報                                  |
| performanceCounters | AppPerformanceCounters | コンピューティング リソースのパフォーマンス測定結果                                  |
| requests            | AppRequests            | アプリケーションで受信された要求                                                     |
| exceptions          | AppExceptions          | アプリケーション ランタイムによってスローされた例外                                  |
| traces              | AppTraces              | TrackTrace() を使って記録されたログ                                                  |

自分の知る限りでは、この仕様に合理性は少なく、クラシック版の影響と考えられますが、将来的にテーブル名が統一されることを期待しています。

# まとめ

- 「Application Insights に保存する」← ×
- 「データは Application Insights に送って監視しています」← ×
- 「Application Insights を使ってデータを収集しています」← 〇
- 「Application Insights で収集して、 Log Analytics ワークスペースに保存しています」← 〇

# 参考資料

- [General availability of workspace-based Application Insights](https://azure.microsoft.com/en-us/updates/general-availability-of-workspacebased-application-insights/)
- [We’re retiring Classic Application Insights on 29 February 2024](https://azure.microsoft.com/en-us/updates/we-re-retiring-classic-application-insights-on-29-february-2024/)
- [Application Insights クラシック版とワークスペース版との違いについて](https://jpazmon-integ.github.io/blog/applicationInsights/aboutDifferentTypesOfAi/)
- [ワークスペースベースの Application Insights リソースに移行する](https://learn.microsoft.com/ja-jp/previous-versions/azure/azure-monitor/app/convert-classic-resource)
- [レガシーとのテーブル構造の違い](https://learn.microsoft.com/ja-jp/previous-versions/azure/azure-monitor/app/convert-classic-resource#table-structure)
