---
title: "【Azure OpenAI】デプロイ毎の月次トークン使用量を監視したい"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [azure, microsoft, azurefunctions, azureopenai, typescript]
published: true
publication_name: "microsoft"
---

# 目的

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

> ↑ のクエリは月のトークン使用量を算出できるものとなっているが、最大粒度（2 日）でラップされてしまうので、アラート ルールは想定の動作をしない。
> これは、設定したクエリと内部で最終的に実行されるクエリが同一ではないことに起因しますが、この confusing な話はまた別の記事で詳しく書こうと思います。

# 解決策

このような（アラート ルールでは目的を達成できない）場合、**Azure Functions** または Logic Apps などを使って自前で監視を組み立てる方法が考えられます。

Logic Apps は Low-Code で便利なのですが、せっかくなので今回は Azure Functions (TypeScript) で監視を実装してみます。

### 今回やること

毎日決まった時間に以下の処理を実行させることを目的とします。

1. メトリック データの取得
2. デプロイメント毎のトークン使用量の算出
3. 値に応じてアラート（今回はメール）を送信

### 今回やらないこと

-   閉域化・権限管理
-   ロバストなコード

# 実装

### 必要なリソース

-   Azure Functions：関数実行
-   Azure Table Stroage：アラートの発報履歴を保存
-   Azure Communication Services：アラート メール送信に使用

### 1．メトリック データの取得

以下の Azure Monitor Metrics API を使ってメトリック データを取得します。

```shell
GET
https://management.azure.com/<AOAI リソース ID>/providers/Microsoft.Insights/metrics?api-version=2023-10-01&metricnames=TokenTransaction&$filter=ModelDeploymentName eq '*'&interval=P1D&timespan=2024-10-01T00:00:00Z/2024-10-25T00:00:00Z
```

https://learn.microsoft.com/ja-jp/rest/api/monitor/metrics/list?view=rest-monitor-2023-10-01&tabs=HTTP

:::message
API の認証には Bearer Token が必要です。以下のコマンドで楽に取得できます。

```
az account get-access-token | jq -r.accessToken
```

:::

:::details レスポンス例

```JSON
{
    "cost": 44639,
    "timespan": "2024-09-30T12:09:00Z/2024-10-31T12:09:00Z",
    "interval": "P1D",
    "value": [
        {
            "id": "<AOAI リソース ID>/providers/Microsoft.Insights/metrics/TokenTransaction",
            "type": "Microsoft.Insights/metrics",
            "name": {
                "value": "TokenTransaction",
                "localizedValue": "Processed Inference Tokens"
            },
            "displayDescription": "Number of inference tokens processed on an OpenAI model. Calculated as prompt tokens (input) plus generated tokens (output). Applies to PTU, PTU-Managed and Pay-as-you-go deployments. To breakdown this metric, you can add a filter or apply splitting by the following dimensions: ModelDeploymentName and ModelName.",
            "unit": "Count",
            "timeseries": [
                {
                    "metadatavalues": [
                        {
                            "name": {
                                "value": "modeldeploymentname",
                                "localizedValue": "modeldeploymentname"
                            },
                            "value": "gpt-4o-momo"
                        }
                    ],
                    "data": [
                        {
                            "timeStamp": "2024-09-30T12:09:00Z",
                            "total": 0
                        },
                        {
                            "timeStamp": "2024-10-01T12:09:00Z",
                            "total": 0
                        },
                        {
                            "timeStamp": "2024-10-02T12:09:00Z",
                            "total": 0
                        },
                        {
                            "timeStamp": "2024-10-03T12:09:00Z",
                            "total": 0
                        },
                        {
                            "timeStamp": "2024-10-04T12:09:00Z",
                            "total": 0
                        },
                        {
                            "timeStamp": "2024-10-05T12:09:00Z",
                            "total": 0
                        },
                        {
                            "timeStamp": "2024-10-06T12:09:00Z",
                            "total": 0
                        },
                        {
                            "timeStamp": "2024-10-07T12:09:00Z",
                            "total": 0
                        },
                        {
                            "timeStamp": "2024-10-08T12:09:00Z",
                            "total": 0
                        },
                        {
                            "timeStamp": "2024-10-09T12:09:00Z",
                            "total": 0
                        },
                        {
                            "timeStamp": "2024-10-10T12:09:00Z",
                            "total": 0
                        },
                        {
                            "timeStamp": "2024-10-11T12:09:00Z",
                            "total": 0
                        },
                        {
                            "timeStamp": "2024-10-12T12:09:00Z",
                            "total": 0
                        },
                        {
                            "timeStamp": "2024-10-13T12:09:00Z",
                            "total": 0
                        },
                        {
                            "timeStamp": "2024-10-14T12:09:00Z",
                            "total": 78
                        },
                        {
                            "timeStamp": "2024-10-15T12:09:00Z",
                            "total": 0
                        },
                        {
                            "timeStamp": "2024-10-16T12:09:00Z",
                            "total": 1183
                        },
                        {
                            "timeStamp": "2024-10-17T12:09:00Z",
                            "total": 0
                        },

                    ]
                }
            ],
            "errorCode": "Success"
        }
    ],
    "namespace": "Microsoft.CognitiveServices/accounts",
    "resourceregion": "eastus"
}
```

:::

### 2．デプロイメント毎のトークン使用量の算出

実際に TypeScript で Azure Functions を作成してみます（コード全体は記事後半に添付）。

```TypeScript
// アクセス トークンを取得
const accessToken = (
    await credential.getToken("https://management.azure.com/.default")
).token;
// 今月のトークン使用量を取得
const timespan = getCurrentMonthTimespan(); // 例：2024-10-01T00:00:00Z/2024-10-28T23:59:59Z
const apiUrl = `https://management.azure.com/${resourceUri}/providers/Microsoft.Insights/metrics?api-version=2023-10-01&metricnames=TokenTransaction&$filter=ModelDeploymentName eq '*'&interval=P1D&timespan=${timespan}`;

const response = await axios.get(apiUrl, {
    headers: {
        Authorization: `Bearer ${accessToken}`,
    },
});

const timeseries = response.data.value[0].timeseries;
const month = getCurrentMonthString();

// 非同期敵に各デプロイメント毎に処理
const promises = timeseries.map(async (deployment) => {
    const deploymentName = deployment.metadatavalues[0].value;
    const data = deployment.data;
    const sumToken = data.reduce((sum, d) => sum + d.total, 0);
    // デプロイメント毎のトークン使用量を評価
    await evaluateTokenUsageForAlert(deploymentName, sumToken, month);
});

await Promise.all(promises);
```

評価の部分（`evaluateTokenUsageForAlert` 関数）は以下のようになります。

```TypeScript
async function evaluateTokenUsageForAlert(
    deploymentName: string,
    sumToken: number,
    month: string
) {
    // Table Storage から「デプロイメント × 月」のユニークなキーでエンティティを取得（例：gpt4dev-2024-10）
    const rowKey = `${deploymentName}-${month}`;
    try {
        let entity: TableEntity | null;

        try {
            entity = await tableClient.getEntity("DeploymentType", rowKey);
        } catch (error) {
            if (error.statusCode === 404) {
                // エンティティが存在しない場合は新規作成
                entity = {
                    partitionKey: "DeploymentType",
                    rowKey: rowKey,
                    SumToken: sumToken,
                    ActionDone: false,
                    LastUpdated: new Date().toISOString(),
                };
                await tableClient.createEntity(entity);
            } else {
                throw error;
            }
        }

        // アラートを送信するかどうかの判定
        if (shouldSendAlert(entity, sumToken)) {
            const result = await sendMail(deploymentName, sumToken);
            if (result === "Succeeded") {
                console.log(`Alert mail sent for ${deploymentName}`);
                entity.ActionDone = true;
                entity.LastUpdated = new Date().toISOString();
                await tableClient.updateEntity(entity);
            } else {
                throw new Error("Failed to send mail");
            }
        } else {
            console.log(`No alert needed for ${deploymentName}`);
        }

        if (sumToken > Number(entity.SumToken)) {
            entity.SumToken = sumToken;
            entity.LastUpdated = new Date().toISOString();
            await tableClient.updateEntity(entity);
        }
    } catch (error) {
        console.error("Error processing deployment", deploymentName, error);
    }
}

// 今月まだそのデプロイメントに関してアラートを送信していないかつ閾値を超えているか判定
function shouldSendAlert(
    entity: TableEntity | null,
    sumToken: number
): boolean {
    return !entity?.ActionDone && sumToken > THRESHOLD;
}
```

### 3．アラート（メール）の送信

Azure Communication Services のメール送信機能を使ってアラートを送信します。
リソースの作り方はアドレスの取得などは今回は省略しますが、以下の記事を参考にして適宜設定してください。

https://zenn.dev/microsoft/articles/azure-communication-service-email
https://qiita.com/ShuntaIto/items/129a802677c81705d481

コードとしては以下のようになります。

```TypeScript
async function sendMail(deploymentName: string, sumToken: number) {
    const emailMessage: EmailMessage = {
        senderAddress: emailSenderAddress,
        recipients: {
            to: [
                {
                    address: emailRecipientAddress,
                },
            ],
        },
        content: {
            // このデプロイメントの今月のトークン使用量が閾値を超えたことを通知するメール
            subject: `[Alert!] ${deploymentName} has exceeded the threshold`,
            plainText: `The deployment ${deploymentName} has exceeded the threshold of ${THRESHOLD} tokens this month. The total token usage is ${sumToken}.`,
        },
    };

    const poller = await client.beginSend(emailMessage);
    const result = await poller.pollUntilDone();
    return result.status;
}

```

全体としてのコードは以下のようになります。

:::details 完全なコード

```typescript
import { app, InvocationContext, Timer } from "@azure/functions";
import { DefaultAzureCredential } from "@azure/identity";
import {
    TableClient,
    AzureNamedKeyCredential,
    odata,
    TableEntity,
} from "@azure/data-tables";
import { EmailClient, EmailMessage } from "@azure/communication-email";
import axios from "axios";

const credential = new DefaultAzureCredential();

const tableClient = new TableClient(
    process.env["TABLE_STORAGE_ENDPOINT"],
    process.env["TABLE_STORAGE_TABLE_NAME"],
    credential
);

const communicationServicesConnectionString =
    process.env["COMMUNICATION_SERVICES_CONNECTION_STRING"];
const client = new EmailClient(communicationServicesConnectionString);

const resourceUri = process.env["AOAI_RESOURCE_URI"];
const THRESHOLD = 1000; // トークン使用量の閾値（仮）

const emailSenderAddress = process.env["EMAIL_SENDER_ADDRESS"];
const emailRecipientAddress = process.env["EMAIL_RECIPIENT_ADDRESS"];

function getCurrentMonthTimespan(): string {
    const today = new Date();
    const startDateTime = new Date(today.getFullYear(), today.getMonth(), 1);
    const startDateTimeISO = startDateTime.toISOString();
    const endDateTimeISO = today.toISOString();
    return `${startDateTimeISO}/${endDateTimeISO}`;
}

function getCurrentMonthString(): string {
    return new Date().toISOString().slice(0, 7);
}

function shouldSendAlert(
    entity: TableEntity | null,
    sumToken: number
): boolean {
    return !entity?.ActionDone && sumToken > THRESHOLD;
}

async function evaluateTokenUsageForAlert(
    deploymentName: string,
    sumToken: number,
    month: string
) {
    // Table Storage から「デプロイメント × 月」のユニークなキーでエンティティを取得（例：gpt4dev-2024-10）
    const rowKey = `${deploymentName}-${month}`;
    try {
        let entity: TableEntity | null;

        try {
            entity = await tableClient.getEntity("DeploymentType", rowKey);
        } catch (error) {
            if (error.statusCode === 404) {
                // エンティティが存在しない場合は新規作成
                entity = {
                    partitionKey: "DeploymentType",
                    rowKey: rowKey,
                    SumToken: sumToken,
                    ActionDone: false,
                    LastUpdated: new Date().toISOString(),
                };
                await tableClient.createEntity(entity);
            } else {
                throw error;
            }
        }

        // アラートを送信するかどうかの判定
        if (shouldSendAlert(entity, sumToken)) {
            const result = await sendMail(deploymentName, sumToken);
            if (result === "Succeeded") {
                console.log(`Alert mail sent for ${deploymentName}`);
                entity.ActionDone = true;
                entity.LastUpdated = new Date().toISOString();
                await tableClient.updateEntity(entity);
            } else {
                throw new Error("Failed to send mail");
            }
        } else {
            console.log(`No alert needed for ${deploymentName}`);
        }

        if (sumToken > Number(entity.SumToken)) {
            entity.SumToken = sumToken;
            entity.LastUpdated = new Date().toISOString();
            await tableClient.updateEntity(entity);
        }
    } catch (error) {
        console.error("Error processing deployment", deploymentName, error);
    }
}

async function sendMail(deploymentName: string, sumToken: number) {
    const emailMessage: EmailMessage = {
        senderAddress: emailSenderAddress,
        recipients: {
            to: [
                {
                    address: emailRecipientAddress,
                },
            ],
        },
        content: {
            // このデプロイメントの今月のトークン使用量が閾値を超えたことを通知するメール
            subject: `[Alert!] ${deploymentName} has exceeded the threshold`,
            plainText: `The deployment ${deploymentName} has exceeded the threshold of ${THRESHOLD} tokens this month. The total token usage is ${sumToken}.`,
        },
    };

    const poller = await client.beginSend(emailMessage);
    const result = await poller.pollUntilDone();
    return result.status;
}

export async function timerTrigger(
    myTimer: Timer,
    context: InvocationContext
): Promise<void> {
    try {
        // アクセス トークンを取得
        const accessToken = (
            await credential.getToken("https://management.azure.com/.default")
        ).token;
        // 今月のトークン使用量を取得
        const timespan = getCurrentMonthTimespan(); // 例：2024-10-01T00:00:00Z/2024-10-28T23:59:59Z
        const apiUrl = `https://management.azure.com/${resourceUri}/providers/Microsoft.Insights/metrics?api-version=2023-10-01&metricnames=TokenTransaction&$filter=ModelDeploymentName eq '*'&interval=P1D&timespan=${timespan}`;

        const response = await axios.get(apiUrl, {
            headers: {
                Authorization: `Bearer ${accessToken}`,
            },
        });

        const timeseries = response.data.value[0].timeseries;
        const month = getCurrentMonthString();

        // 非同期敵に各デプロイメント毎に処理
        const promises = timeseries.map(async (deployment) => {
            const deploymentName = deployment.metadatavalues[0].value;
            const data = deployment.data;

            const sumToken = data.reduce((sum, d) => sum + d.total, 0);

            await evaluateTokenUsageForAlert(deploymentName, sumToken, month);
        });

        await Promise.all(promises);

        context.log("Success");
    } catch (error) {
        context.log("Error", error);
    }
}

app.timer("timerTrigger", {
    // 平日の毎日午前 9 時に実行
    schedule: "0 0 9 * * 1-5",
    handler: timerTrigger,
});
```

:::

### ローカルでの動かしてみる

ローカルで動作確認する場合は、HTTP トリガーに変換して以下のように実行すると楽です。

```
npm run build
npm run start (func start)
```

:::message alert
ローカルでユーザー権限でクレデンシャルをつかってテーブルクライアントを作成する場合、ユーザーに「ストレージ テーブル データ共同作成者」のロールを割り当てる必要があります。
:::

Table Storage にエンティティが作成されており、`gpt-4o-momo` というデプロイメントのトークン使用量が 1000 を超えており、アラートメールが送信されたようです。
![alt text](/images/monitor-monthly-token-usage-by-deployments/table-storage-ss.png)

きちんとアラートメールが送信されていました。
![alt text](/images/monitor-monthly-token-usage-by-deployments/mail-sent-me.png)

# まとめ

今回は、Azure Functions を使って Azure OpenAI リソースについて、月毎のトークン使用量をデプロイメント毎に監視する方法を紹介しました。その中で Azure Communication Service と Azure Table Storage を使ってアラートの送信と履歴の保存・管理を行うところまでを実装しました。

この記事を参考に、Azure OpenAI リソースのトークン使用量を監視する際に、アラート ルールでは実現できない場合に、Azure Functions などを使って自前で監視を組み立てる方法を検討してみてください。
