---
title: "【Azure のアラート設計ベースライン】Azure Monitor Baseline Alerts の紹介"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [azure, microsoft, azuremonitor, monitor]
published: true
publication_name: "microsoft"
---

# はじめに

アラートって大事ですよね。システムが突然異常を起こしたとき、素早くキャッチして対応するためにアラートは欠かせません。でも、実際「どういうアラートを作ればいいの？」と悩んでしまうことも多いのではないでしょうか？

そこで今回ご紹介するのが、[**Azure Monitor Baseline Alerts**](https://azure.github.io/azure-monitor-baseline-alerts/welcome/) です。

# Azure Monitor Baseline Alerts とは

Azure Monitor Baseline Alerts (AMBA) は、各 Azure リソースに対して最適なアラートルールの推奨値を提示するツールです。  
単なる閾値の設定だけでなく、**評価頻度**、**評価期間**、**緊急度** なども含めた詳細なガイドラインを提供しており、Azure 環境の監視・運用をより効率的かつプロアクティブに実施するための強力なサポートとなります。

この記事では、AMBA の概要と、実際に提示されている推奨アラートの例、さらにアラートルールのデプロイに関するテンプレートやパッケージについて解説します。

---

## 推奨アラート例

AMBA では、各リソースに対して推奨されるアラートルールが提供されています。  
例えば、**仮想マシン (VM)** に関するアラートの例を以下に示します。

::: message
[**VM の推奨アラート一覧**](https://azure.github.io/azure-monitor-baseline-alerts/services/Compute/virtualMachines/)
:::

### 例 1: Available Memory Bytes に基づく静的閾値アラート

リンク: [Available Memory Bytes の推奨アラート](https://azure.github.io/azure-monitor-baseline-alerts/services/Compute/virtualMachines/#available-memory-bytes)

| プロパティ          | 値                                |
| ------------------- | --------------------------------- |
| criterionType       | StaticThresholdCriterion          |
| evaluationFrequency | PT5M                              |
| metricName          | Available Memory Bytes            |
| metricNamespace     | Microsoft.Compute/virtualMachines |
| operator            | LessThan                          |
| severity            | 3                                 |
| threshold           | 1000000000                        |
| timeAggregation     | Average                           |
| windowSize          | PT5M                              |

このルールは、指定された評価期間内に「Available Memory Bytes」が 1,000 MB を下回る場合にアラートを発生させます。

---

### 例 2: Available Memory Percentage に基づくログ アラート

リンク: [Available Memory Percentage の推奨アラート](https://azure.github.io/azure-monitor-baseline-alerts/services/Compute/virtualMachines/#available-memory-percentage)

この例では、メトリックとして直接収集されていないメモリ使用率を、使用量とトータル量をもとに計算するクエリが提示されています。

#### 基本プロパティ

| プロパティ          | 値              |
| ------------------- | --------------- |
| autoMitigate        | true            |
| autoResolve         | true            |
| autoResolveTime     | 0:10:00         |
| evaluationFrequency | PT5M            |
| metricMeasureColumn | AggregatedValue |
| operator            | LessThan        |
| resouceIdColumn     | \_ResourceId    |
| severity            | 2               |
| threshold           | 10              |
| timeAggregation     | Average         |
| windowSize          | PT15M           |

#### Dimensions と Failing Periods

-   **dimensions**:

    ```json
    [
        {
            "name": "Computer",
            "operator": "Include",
            "values": ["*"]
        }
    ]
    ```

-   **failingPeriods**:
    ```json
    {
        "minFailingPeriodsToAlert": 1,
        "numberOfEvaluationPeriods": 1
    }
    ```

#### クエリ

以下のクエリは、`InsightsMetrics` テーブルからメモリに関するデータを抽出し、利用可能なメモリの割合を算出しています。

```kusto
InsightsMetrics
| where Origin == "vm.azm.ms"
| where Namespace == "Memory" and Name == "AvailableMB"
| extend TotalMemory = toreal(todynamic(Tags)["vm.azm.ms/memorySizeMB"])
| extend AvailableMemoryPercentage = (toreal(Val) / TotalMemory) \* 100.0
| summarize AggregatedValue = avg(AvailableMemoryPercentage) by bin(TimeGenerated, 15m), Computer, \_ResourceId
```

このルールは、算出された利用可能メモリの平均値が 10% を下回る場合にアラートを発生させます。

# Infrastructure as Code (IaC) テンプレート

AMBA では、上記のアラートルール設定とともに、それを簡単にデプロイできるよう、さまざまな **IaC テンプレート**が提供されています。主な提供テンプレートは以下の通りです。

-   Azure ポータルでのカスタム デプロイ ボタン
    -   アラートルール単体のデプロイテンプレート
    -   アラートルールの存在や必要に応じた自動展開を監査・強制する Azure Policy のデプロイテンプレート
        ![alt text](/images/introduce-amba/templates.png)
-   ARM テンプレート
    :::details テンプレートを見る
    ```json
    {
        "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
        "contentVersion": "1.0.0.0",
        "parameters": {
            "alertName": {
                "type": "string",
                "minLength": 1,
                "metadata": {
                    "description": "Name of the alert"
                }
            },
            "alertDescription": {
                "type": "string",
                "defaultValue": "Amount of physical memory, in bytes, immediately available for allocation to a process or for system use in the Virtual Machine",
                "metadata": {
                    "description": "Description of alert"
                }
            },
            "targetResourceId": {
                "type": "string",
                "minLength": 1,
                "metadata": {
                    "description": "List of Azure resource Ids seperated by a comma. For example - /subscriptions/00000000-0000-0000-0000-0000-00000000/resourceGroup/resource-group-name/Microsoft.compute/virtualMachines/vm-name"
                }
            },
            "targetResourceRegion": {
                "type": "string",
                "metadata": {
                    "description": "Azure region in which target resources to be monitored are in (without spaces). For example: EastUS"
                }
            },
            "targetResourceType": {
                "type": "string",
                "minLength": 1,
                "metadata": {
                    "description": "Resource type of target resources to be monitored."
                }
            },
            "isEnabled": {
                "type": "bool",
                "defaultValue": true,
                "metadata": {
                    "description": "Specifies whether the alert is enabled"
                }
            },
            "alertSeverity": {
                "type": "int",
                "defaultValue": 3,
                "allowedValues": [0, 1, 2, 3, 4],
                "metadata": {
                    "description": "Severity of alert {0,1,2,3,4}"
                }
            },
            "operator": {
                "type": "string",
                "defaultValue": "LessThan",
                "allowedValues": [
                    "Equals",
                    "GreaterThan",
                    "GreaterThanOrEqual",
                    "LessThan",
                    "LessThanOrEqual"
                ],
                "metadata": {
                    "description": "Operator comparing the current value with the threshold value."
                }
            },
            "threshold": {
                "type": "string",
                "defaultValue": "1000000000",
                "metadata": {
                    "description": "The threshold value at which the alert is activated."
                }
            },
            "timeAggregation": {
                "type": "string",
                "defaultValue": "Average",
                "allowedValues": [
                    "Average",
                    "Minimum",
                    "Maximum",
                    "Total",
                    "Count"
                ],
                "metadata": {
                    "description": "How the data that is collected should be combined over time."
                }
            },
            "windowSize": {
                "type": "string",
                "defaultValue": "PT5M",
                "allowedValues": [
                    "PT1M",
                    "PT5M",
                    "PT15M",
                    "PT30M",
                    "PT1H",
                    "PT6H",
                    "PT12H",
                    "PT24H",
                    "PT1D"
                ],
                "metadata": {
                    "description": "Period of time used to monitor alert activity based on the threshold. Must be between one minute and one day. ISO 8601 duration format."
                }
            },
            "evaluationFrequency": {
                "type": "string",
                "defaultValue": "PT5M",
                "allowedValues": ["PT1M", "PT5M", "PT15M", "PT30M", "PT1H"],
                "metadata": {
                    "description": "how often the metric alert is evaluated represented in ISO 8601 duration format"
                }
            },
            "currentDateTimeUtcNow": {
                "type": "string",
                "defaultValue": "[utcNow()]",
                "metadata": {
                    "description": "The current date and time using the utcNow function. Used for deployment name uniqueness"
                }
            },
            "telemetryOptOut": {
                "type": "string",
                "defaultValue": "No",
                "allowedValues": ["Yes", "No"],
                "metadata": {
                    "description": "The customer usage identifier used for telemetry purposes. The default value of False enables telemetry. The value of True disables telemetry."
                }
            }
        },
        "variables": {
            "pidDeploymentName": "[take(concat('pid-8bb7cf8a-bcf7-4264-abcb-703ace2fc84d-', uniqueString(resourceGroup().id, parameters('alertName'), parameters('currentDateTimeUtcNow'))), 64)]",
            "varTargetResourceId": "[split(parameters('targetResourceId'), ',')]"
        },
        "resources": [
            {
                "type": "Microsoft.Insights/metricAlerts",
                "apiVersion": "2018-03-01",
                "name": "[parameters('alertName')]",
                "location": "global",
                "tags": {
                    "_deployed_by_amba": true
                },
                "properties": {
                    "description": "[parameters('alertDescription')]",
                    "scopes": "[variables('varTargetResourceId')]",
                    "targetResourceType": "[parameters('targetResourceType')]",
                    "targetResourceRegion": "[parameters('targetResourceRegion')]",
                    "severity": "[parameters('alertSeverity')]",
                    "enabled": "[parameters('isEnabled')]",
                    "evaluationFrequency": "[parameters('evaluationFrequency')]",
                    "windowSize": "[parameters('windowSize')]",
                    "criteria": {
                        "odata.type": "Microsoft.Azure.Monitor.MultipleResourceMultipleMetricCriteria",
                        "allOf": [
                            {
                                "name": "1st criterion",
                                "metricName": "Available Memory Bytes",
                                "dimensions": [],
                                "operator": "[parameters('operator')]",
                                "threshold": "[parameters('threshold')]",
                                "timeAggregation": "[parameters('timeAggregation')]",
                                "criterionType": "StaticThresholdCriterion"
                            }
                        ]
                    }
                }
            },
            {
                "condition": "[equals(parameters('telemetryOptOut'), 'No')]",
                "apiVersion": "2023-07-01",
                "name": "[variables('pidDeploymentName')]",
                "type": "Microsoft.Resources/deployments",
                "properties": {
                    "mode": "Incremental",
                    "template": {
                        "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
                        "contentVersion": "1.0.0.0",
                        "resources": []
                    }
                }
            }
        ]
    }
    ```
    :::
-   Bicep テンプレート
    :::details テンプレートを見る

    ```bicep
    @description('Name of the alert')
    @minLength(1)
    param alertName string

    @description('Description of alert')
    param alertDescription string = 'Amount of physical memory, in bytes, immediately available for allocation to a process or for system use in the Virtual Machine'

    @description('Array of Azure resource Ids. For example - /subscriptions/00000000-0000-0000-0000-0000-00000000/resourceGroup/resource-group-name/Microsoft.compute/virtualMachines/vm-name')
    @minLength(1)
    param targetResourceId array

    @description('Azure region in which target resources to be monitored are in (without spaces). For example: EastUS')
    param targetResourceRegion string

    @description('Resource type of target resources to be monitored.')
    @minLength(1)
    param targetResourceType string

    @description('Specifies whether the alert is enabled')
    param isEnabled bool = true

    @description('Severity of alert {0,1,2,3,4}')
    @allowed([
    0
    1
    2
    3
    4
    ])
    param alertSeverity int = 3

    @description('Operator comparing the current value with the threshold value.')
    @allowed([
    'Equals'
    'GreaterThan'
    'GreaterThanOrEqual'
    'LessThan'
    'LessThanOrEqual'
    ])
    param operator string = 'LessThan'

    @description('The threshold value at which the alert is activated.')
    param threshold int = 1000000000

    @description('How the data that is collected should be combined over time.')
    @allowed([
    'Average'
    'Minimum'
    'Maximum'
    'Total'
    'Count'
    ])
    param timeAggregation string = 'Average'

    @description('Period of time used to monitor alert activity based on the threshold. Must be between one minute and one day. ISO 8601 duration format.')
    @allowed([
    'PT1M'
    'PT5M'
    'PT15M'
    'PT30M'
    'PT1H'
    'PT6H'
    'PT12H'
    'PT24H'
    'P1D'
    ])
    param windowSize string = 'PT5M'

    @description('how often the metric alert is evaluated represented in ISO 8601 duration format')
    @allowed([
    'PT1M'
    'PT5M'
    'PT15M'
    'PT30M'
    'PT1H'
    ])
    param evaluationFrequency string = 'PT5M'

    @description('"The current date and time using the utcNow function. Used for deployment name uniqueness')
    param currentDateTimeUtcNow string = utcNow()

    @description('The customer usage identifier used for telemetry purposes. The default value of False enables telemetry. The value of True disables telemetry.')
    @allowed([
    'Yes'
    'No'
    ])
    param telemetryOptOut string = 'No'

    resource metricAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
    name: alertName
    location: 'global'
    tags: {
        _deployed_by_amba: 'true'
    }
    properties: {
        description: alertDescription
        scopes: targetResourceId
        targetResourceType: targetResourceType
        targetResourceRegion: targetResourceRegion
        severity: alertSeverity
        enabled: isEnabled
        evaluationFrequency: evaluationFrequency
        windowSize: windowSize
        criteria: {
        'odata.type': 'Microsoft.Azure.Monitor.MultipleResourceMultipleMetricCriteria'
        allOf: [
            {
            name: '1st criterion'
            metricName: 'Available Memory Bytes'
            dimensions: [[]]
            operator: operator
            threshold: threshold
            timeAggregation: timeAggregation
            criterionType: 'StaticThresholdCriterion'
            }
        ]
        }
    }
    }

    var ambaTelemetryPidName = 'pid-8bb7cf8a-bcf7-4264-abcb-703ace2fc84d-${uniqueString(resourceGroup().id, alertName, currentDateTimeUtcNow)}'
    resource ambaTelemetryPid 'Microsoft.Resources/deployments@2023-07-01' =  if (telemetryOptOut == 'No') {
    name: ambaTelemetryPidName
    tags: {
        _deployed_by_amba: 'true'
    }
    properties: {
        mode: 'Incremental'
        template: {
        '$schema': 'https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#'
        contentVersion: '1.0.0.0'
        resources: []
        }
    }
    }
    ```

    :::

-   Azure Policy 定義 (JSON)
    :::details テンプレートを見る
    ```json
    {
        "mode": "All",
        "parameters": {
            "severity": {
                "type": "String",
                "metadata": {
                    "displayName": "Severity",
                    "description": "Severity of the Alert"
                },
                "allowedValues": ["0", "1", "2", "3", "4"],
                "defaultValue": "3"
            },
            "windowSize": {
                "type": "String",
                "metadata": {
                    "displayName": "Window Size",
                    "description": "Window size for the alert"
                },
                "allowedValues": [
                    "PT1M",
                    "PT5M",
                    "PT15M",
                    "PT30M",
                    "PT1H",
                    "PT6H",
                    "PT12H",
                    "P1D"
                ],
                "defaultValue": "PT5M"
            },
            "evaluationFrequency": {
                "type": "String",
                "metadata": {
                    "displayName": "Evaluation Frequency",
                    "description": "Evaluation frequency for the alert"
                },
                "allowedValues": ["PT1M", "PT5M", "PT15M", "PT30M", "PT1H"],
                "defaultValue": "PT5M"
            },
            "autoMitigate": {
                "type": "String",
                "metadata": {
                    "displayName": "Auto Mitigate",
                    "description": "Auto Mitigate for the alert"
                },
                "allowedValues": ["true", "false"],
                "defaultValue": "true"
            },
            "enabled": {
                "type": "String",
                "metadata": {
                    "displayName": "Alert State",
                    "description": "Alert state for the alert"
                },
                "allowedValues": ["true", "false"],
                "defaultValue": "true"
            },
            "threshold": {
                "type": "String",
                "metadata": {
                    "displayName": "Threshold",
                    "description": "Threshold for the alert"
                },
                "defaultValue": "1000000000"
            },
            "effect": {
                "type": "String",
                "metadata": {
                    "displayName": "Effect",
                    "description": "Effect of the policy"
                },
                "allowedValues": ["deployIfNotExists", "disabled"],
                "defaultValue": "deployIfNotExists"
            },
            "MonitorDisableTagName": {
                "type": "String",
                "metadata": {
                    "displayName": "Monitoring disabled tag name",
                    "description": "Tag name used to disable monitoring at the resource level. Set to true if monitoring should be disabled."
                },
                "defaultValue": "MonitorDisable"
            },
            "MonitorDisableTagValues": {
                "type": "Array",
                "metadata": {
                    "displayName": "Monitoring disabled tag values(s)",
                    "description": "Tag value(s) used to disable monitoring at the resource level. Set to true if monitoring should be disabled."
                },
                "defaultValue": ["true", "Test", "Dev", "Sandbox"]
            }
        },
        "policyRule": {
            "if": {
                "allOf": [
                    {
                        "field": "type",
                        "equals": "Microsoft.Compute/virtualMachines"
                    },
                    {
                        "field": "[concat('tags[', parameters('MonitorDisableTagName'), ']')]",
                        "notIn": "[parameters('MonitorDisableTagValues')]"
                    }
                ]
            },
            "then": {
                "effect": "[parameters('effect')]",
                "details": {
                    "roleDefinitionIds": [
                        "/providers/Microsoft.Authorization/roleDefinitions/b24988ac-6180-42a0-ab88-20f7382dd24c"
                    ],
                    "type": "Microsoft.Insights/metricAlerts",
                    "existenceCondition": {
                        "allOf": [
                            {
                                "field": "Microsoft.Insights/metricAlerts/criteria.Microsoft.Azure.Monitor.MultipleResourceMultipleMetricCriteria.allOf[*].metricNamespace",
                                "equals": "Microsoft.Compute/virtualMachines"
                            },
                            {
                                "field": "Microsoft.Insights/metricAlerts/criteria.Microsoft.Azure.Monitor.MultipleResourceMultipleMetricCriteria.allOf[*].metricName",
                                "equals": "Available Memory Bytes"
                            },
                            {
                                "field": "Microsoft.Insights/metricalerts/scopes[*]",
                                "equals": "[concat(subscription().id, '/resourceGroups/', resourceGroup().name, '/providers/Microsoft.Compute/virtualMachines/', field('fullName'))]"
                            },
                            {
                                "field": "Microsoft.Insights/metricAlerts/enabled",
                                "equals": "[parameters('enabled')]"
                            },
                            {
                                "field": "Microsoft.Insights/metricAlerts/evaluationFrequency",
                                "equals": "[parameters('evaluationFrequency')]"
                            },
                            {
                                "field": "Microsoft.Insights/metricAlerts/windowSize",
                                "equals": "[parameters('windowSize')]"
                            },
                            {
                                "field": "Microsoft.Insights/metricalerts/severity",
                                "equals": "[parameters('severity')]"
                            },
                            {
                                "field": "Microsoft.Insights/metricAlerts/autoMitigate",
                                "equals": "[parameters('autoMitigate')]"
                            },
                            {
                                "field": "Microsoft.Insights/metricAlerts/criteria.Microsoft-Azure-Monitor-SingleResourceMultipleMetricCriteria.allOf[*].timeAggregation",
                                "equals": "Average"
                            },
                            {
                                "field": "Microsoft.Insights/metricAlerts/criteria.Microsoft.Azure.Monitor.MultipleResourceMultipleMetricCriteria.allOf[*].StaticThresholdCriterion.operator",
                                "equals": "LessThan"
                            },
                            {
                                "field": "Microsoft.Insights/metricAlerts/criteria.Microsoft.Azure.Monitor.MultipleResourceMultipleMetricCriteria.allOf[*].StaticThresholdCriterion.threshold",
                                "equals": "[if(contains(field('tags'), '_amba-Available Memory Bytes-threshold-Override_'), field('tags._amba-Available Memory Bytes-threshold-Override_'), parameters('threshold'))]"
                            }
                        ]
                    },
                    "deployment": {
                        "properties": {
                            "mode": "incremental",
                            "template": {
                                "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
                                "contentVersion": "1.0.0.0",
                                "parameters": {
                                    "resourceName": {
                                        "type": "String",
                                        "metadata": {
                                            "displayName": "resourceName",
                                            "description": "Name of the resource"
                                        }
                                    },
                                    "resourceId": {
                                        "type": "String",
                                        "metadata": {
                                            "displayName": "resourceId",
                                            "description": "Resource ID of the resource emitting the metric that will be used for the comparison"
                                        }
                                    },
                                    "severity": {
                                        "type": "String"
                                    },
                                    "windowSize": {
                                        "type": "String"
                                    },
                                    "evaluationFrequency": {
                                        "type": "String"
                                    },
                                    "autoMitigate": {
                                        "type": "String"
                                    },
                                    "enabled": {
                                        "type": "String"
                                    },
                                    "threshold": {
                                        "type": "String"
                                    }
                                },
                                "variables": {},
                                "resources": [
                                    {
                                        "type": "Microsoft.Insights/metricAlerts",
                                        "apiVersion": "2018-03-01",
                                        "name": "[concat(parameters('resourceName'), '-Available Memory Bytes')]",
                                        "location": "global",
                                        "tags": {
                                            "_deployed_by_amba": true
                                        },
                                        "properties": {
                                            "description": "Metric Alert for Compute virtualMachines Available Memory Bytes",
                                            "severity": "[parameters('severity')]",
                                            "enabled": "[parameters('enabled')]",
                                            "scopes": [
                                                "[parameters('resourceId')]"
                                            ],
                                            "evaluationFrequency": "[parameters('evaluationFrequency')]",
                                            "windowSize": "[parameters('windowSize')]",
                                            "criteria": {
                                                "allOf": [
                                                    {
                                                        "name": "Available Memory Bytes",
                                                        "metricNamespace": "Microsoft.Compute/virtualMachines",
                                                        "metricName": "Available Memory Bytes",
                                                        "operator": "LessThan",
                                                        "threshold": "[parameters('threshold')]",
                                                        "timeAggregation": "Average",
                                                        "criterionType": "StaticThresholdCriterion"
                                                    }
                                                ],
                                                "odata.type": "Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria"
                                            },
                                            "autoMitigate": "[parameters('autoMitigate')]",
                                            "parameters": {
                                                "severity": {
                                                    "value": "[parameters('severity')]"
                                                },
                                                "windowSize": {
                                                    "value": "[parameters('windowSize')]"
                                                },
                                                "evaluationFrequency": {
                                                    "value": "[parameters('evaluationFrequency')]"
                                                },
                                                "autoMitigate": {
                                                    "value": "[parameters('autoMitigate')]"
                                                },
                                                "enabled": {
                                                    "value": "[parameters('enabled')]"
                                                },
                                                "threshold": {
                                                    "value": "[parameters('threshold')]"
                                                }
                                            }
                                        }
                                    }
                                ]
                            },
                            "parameters": {
                                "resourceName": {
                                    "value": "[field('name')]"
                                },
                                "resourceId": {
                                    "value": "[field('id')]"
                                },
                                "severity": {
                                    "value": "[parameters('severity')]"
                                },
                                "windowSize": {
                                    "value": "[parameters('windowSize')]"
                                },
                                "evaluationFrequency": {
                                    "value": "[parameters('evaluationFrequency')]"
                                },
                                "autoMitigate": {
                                    "value": "[parameters('autoMitigate')]"
                                },
                                "enabled": {
                                    "value": "[parameters('enabled')]"
                                },
                                "threshold": {
                                    "value": "[if(contains(field('tags'), '_amba-Available Memory Bytes-threshold-Override_'), field('tags._amba-Available Memory Bytes-threshold-Override_'), parameters('threshold'))]"
                                }
                            }
                        }
                    }
                }
            }
        }
    }
    ```
    :::

これにより、組織内で一貫したアラートルールの展開と管理が容易になります。

# Monitoring Packs

また、よくあるパターンやシナリオに対して、アラートルール一式をパッケージ化した Monitoring Packs も提供されています。各パックは以下のリンクから確認できます。

-   [PaaS Monitoring Packs](https://azure.github.io/azure-monitor-baseline-alerts/patterns/monitoring-packs/PaaS/)
-   [IaaS Monitoring Packs](https://azure.github.io/azure-monitor-baseline-alerts/patterns/monitoring-packs/IaaS/)
-   [Platform Monitoring Packs](https://azure.github.io/azure-monitor-baseline-alerts/patterns/monitoring-packs/Platform/)

![alt text](/images/introduce-amba/paas-pack.png)

これらのパックを活用することで、さまざまなシナリオに対して必要なアラートを一括して設定することができます。

# まとめ

Azure Monitor Baseline Alerts は、Azure 環境におけるアラート設計のベースラインを提供することで、運用の自動化と一貫性の確保を実現します。
各種アラートルールの推奨値や、IaC テンプレート、Monitoring Packs を活用し、プロアクティブな監視体制を構築することが可能です。

# 参考資料

-   [Azure Monitor Baseline Alerts](https://azure.github.io/azure-monitor-baseline-alerts/welcome/)
-   [Azure Monitor アラートのベスト プラクティス](https://learn.microsoft.com/ja-jp/azure/azure-monitor/best-practices-alerts)
