---
title: "ログ検索アラート ルールのためのカスタム ログ挿入方法"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [azure, azuremonitor, monitor, loganalytics]
published: true
publication_name: "microsoft"
---

# 概要

ログ検索アラート ルールの動作を確認したり、テストする時どうしていますか？アプリを作ってログを出力して確認している方もいるかもしれません。しかし、その方法だとあまり効率的ではありません。

そこでこの記事では、Logic Apps を使って任意のログを任意のタイミングで出力する方法を紹介します。これにより、ログ検索アラート ルールの動作を確認する際に、より効率的にテストができるようになります。

# 登場人物

-   Logic Apps
-   Log Analytics ワークスペース
-   データ収集ルール
-   データ収集エンドポイント
-   アラート ルール

# 0. Log Analytics ワークスペース作成

ログの保存先となる Log Analytics ワークスペースを作成しておきます（既存のワークスペースを使っても構いません）。
![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/create-law.png)

# 1. Logic Apps

## リソース作成

Azure ポータルから Logic Apps を作成します。特に必要な設定は無いですが、今回はホスティング プランを「従量課金」にしました。

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/create-logic-app.png)

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/create-logic-app2.png)

## Logic Apps 設定

## ワークフロー定義

作成した Logic Apps リソースの「ロジック アプリ コード レビュー」ブレードに移動し、以下の JSON を貼り付けます。

:::message
この JSON は、1 時間に 1 回、指定したエンドポイントにデータを送信する Logic Apps の定義です。`<ENDPOINT_URI>` には、後述するデータ収集エンドポイントの URI を指定します。

参考: [Azure Logic Apps でロジック アプリ ワークフロー定義の JSON を作成、編集、拡張する](https://learn.microsoft.com/ja-jp/azure/logic-apps/logic-apps-author-definitions)
:::

```JSON
{
    "definition": {
        "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
        "actions": {
            "HTTP_2": {
                "inputs": {
                    "authentication": {
                        "audience": "https://monitor.azure.com",
                        "type": "ManagedServiceIdentity"
                    },
                    "body": [
                        {
                            "RawData": "Test"
                        }
                    ],
                    "headers": {
                        "Content-Type": "application/json"
                    },
                    "method": "POST",
                    "uri": "<ENDPOINT_URI>"
                },
                "runAfter": {},
                "runtimeConfiguration": {
                    "contentTransfer": {
                        "transferMode": "Chunked"
                    }
                },
                "type": "Http"
            }
        },
        "contentVersion": "1.0.0.0",
        "outputs": {},
        "parameters": {
            "$connections": {
                "defaultValue": {},
                "type": "Object"
            }
        },
        "staticResults": {
            "HTTP0": {
                "code": "200",
                "outputs": {
                    "statusCode": "OK"
                },
                "status": "Succeeded"
            }
        },
        "triggers": {
            "Recurrence": {
                "evaluatedRecurrence": {
                    "frequency": "Hour",
                    "interval": 1
                },
                "recurrence": {
                    "frequency": "Hour",
                    "interval": 1
                },
                "type": "Recurrence"
            }
        }
    },
    "parameters": {
        "$connections": {
            "value": {}
        }
    }
}

```

## マネージド ID の有効化

JSON でワークフロー定義を作成した後に「ロジック アプリ デザイナー」ブレードに移動し、「HTTP 2」アクションを選択すると、マネージド ID が無効になっている旨のメッセージが表示されます。

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/disabled-managed-id.png)

「ID」ブレードに移動し、システム割り当てマネージド ID を有効にします。

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/enabled-managed-id.png)

このマネージド ID に「[監視メトリック発行者](https://learn.microsoft.com/ja-jp/azure/role-based-access-control/built-in-roles/monitor#monitoring-metrics-publisher)」ロールを割り当てます。これによって Logic Apps が Log Analytics ワークスペースにデータを送信できるようになります。

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/add-role.png)

# 2. データ収集エンドポイント作成

データ収集エンドポイントを作成します。このエンドポイントは、この後、作成するカスタム ログ テーブルにデータを送信するために使用します。

:::message
データ収集エンドポイントと後述するデータ収集ルールについての概要は、以下の公式ブログが参考になります（ブログを執筆した徳田さんは私の同期です）。

参考: [データ収集ルール作成時のデータ収集エンドポイントの構成について](https://jpazmon-integ.github.io/blog/LogAnalytics/DataCollectionEndpoint/)

:::

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/create-dce.png)

# 3. データ収集ルール作成

データ収集ルールを作成します。このルールは、データ収集エンドポイントからデータを受信し、指定したテーブルにデータを格納します。

Log Analytics ワークスペース リソースの「テーブル」ブレードに移動し、「新しいカスタム ログ（DCR ベース）」を選択して、カスタム ログ テーブルとそれに紐づくデータ収集ルールを作成していきます。

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/create-dcr.png)

次の画面で、カスタム ログのサンプル スキーマとして以下の JSON ファイルをアップロードします。

```JSON
[{ "RawData": "aaa" }]
```

アップロードしたスキーマが正しく読み込まれると、以下のようにスキーマが表示されます。

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/add-sample-schema.png)

確認して良ければ「作成」をクリックしてカスタム ログ テーブルを作成します。

# 4. アプリ設定

作成したデータ収集エンドポイントとデータ収集ルールの情報を Logic Apps のワークフロー定義に追加します。

## データ収集エンドポイント

Azure ポータルからデータ収集エンドポイントのログ インジェスト エンドポイントを取得します。

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/dce-endpoint.png)

## データ収集ルール

Azure ポータルからデータ収集ルールの JSON ビューを選択し、`immutableId` と `dataFlows.streams` の値を取得します。

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/dcr-jsonview.png)

## Logic Apps

「ロジック アプリ デザイナー」にて、「HTTP 2」アクションの URI を以下の値に更新し保存します。

```
<ログ インジェスト エンドポイント>/dataCollectionRules/<immutableID>/streams/<dataFlows.streams>?api-version=2023-01-01
```

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/set-http-uri.png)

# 5. ログ出力

そのまま「ロジック アプリ デザイナー」ブレード内で「実行」を選択し、Logic Apps のワークフローをテスト実行します。

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/log-ingest-test.png)

「実行履歴」ブレードに移動し、実行が成功したことを確認します。

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/log-ingest-succeeded.png)

Log Analytics ワークスペースにデータが格納されていることを確認します。

:::message
Log Analytics ワークスペースにてデータが確認できるようになるまでには、数分かかる場合があります。この遅延の要因については以下の公式ドキュメントに記載されています。

参考: [Azure Monitor でのログ データ インジェスト時間](https://learn.microsoft.com/ja-jp/azure/azure-monitor/logs/data-ingestion-time)
:::

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/kusto-result.png)

# (Optional) 6. ログ検索アラート ルールのテスト

作成したカスタム ログ テーブルを使って、ログ検索アラート ルールを作成し、テストします。

スコープには、作成したカスタム ログ テーブルがある Log Analytics ワークスペースを指定します。

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/alert-rule-scope.png)

条件には、カスタム ログ テーブルに格納されたデータを検出する条件を指定します。

> 以下画像内のルールを簡潔にまとめると、`RawData` フィールドに `Test` という文字列が含まれるログが直近 1 時間以内に 1 つ以上あるかどうかを 5 分ごとにチェックするルールです。

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/alert-rule-condition.png)

アクションには、アラートが発生した際に通知を送信する設定を行います。今回はメール通知を設定しました。

以下のようにアクション グループを作成し、アクション グループにメール通知を設定します。

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/create-action-group.png)

通知方法としてメールを選択し、通知先のメールアドレスを入力します。

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/create-action-group-2.png)

アクションはスキップして、アクション グループを作成します。

アクション グループを作成すると、アラート ルールの作成画面に戻り、自動的に作成したアクション グループが選択されていることを確認します。

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/after-create-ag.png)

「詳細」タブでアラート ルールの名前を入力し、アラート ルールを作成します。

少し時間を置いて、アラート ルールが発報することを確認します。

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/fired-alert-log.png)

アクションとして設定したメールアドレスに通知が届いていることを確認します。

![alt text](/images/custom-log-ingestion-for-validate-log-alert-rule/fired-alert-mail.png)

# まとめ

この記事では、Logic Apps を使って任意のログを任意のタイミングで出力する方法を紹介しました。これにより、ログ検索アラート ルールの動作を確認する際に、より効率的にテストができるようになります。

# 参考資料

-   [Azure Logic Apps でロジック アプリ ワークフロー定義の JSON を作成、編集、拡張する](https://learn.microsoft.com/ja-jp/azure/logic-apps/logic-apps-author-definitions)
-   [データ収集ルール作成時のデータ収集エンドポイントの構成について](https://jpazmon-integ.github.io/blog/LogAnalytics/DataCollectionEndpoint/)
-   [Azure Monitor でのデータ収集ルールの構造](https://learn.microsoft.com/ja-jp/azure/azure-monitor/essentials/data-collection-rule-structure)
-   [Azure Monitor でのログ データ インジェスト時間](https://learn.microsoft.com/ja-jp/azure/azure-monitor/logs/data-ingestion-time)
