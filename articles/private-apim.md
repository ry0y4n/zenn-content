---
title: "【API Management の閉域化】プライベートな API ゲートウェイを構築しよう！"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure", "azureapimanagement", "api"]
published: true
publication_name: "microsoft"
---

~~いきなりですが、2025 年 4 月 10 日、**API Management の Standard v2 プランにおいて、プライベート エンドポイントのパブリック プレビューが開始されました！** これによって、今までよりかなり安価に API Management を閉域化することができるようになりました！~~

> ~~参考：[Announcing open public preview of inbound private endpoint for Standard v2 tier of API Management](https://techcommunity.microsoft.com/blog/integrationsonazureblog/announcing-open-public-preview-of-inbound-private-endpoint-for-standard-v2-tier-/4402521)~~

> 追記：2025 年 5 月 20 日、**API Management の Standard v2 プランにおいて、プライベート エンドポイントの「一般提供（GA）」が開始されました！**（参考：[GA: Inbound private endpoint for Standard v2 tier of Azure API Management](https://techcommunity.microsoft.com/blog/integrationsonazureblog/ga-inbound-private-endpoint-for-standard-v2-tier-of-azure-api-management/4415374)）

プライベート エンドポイントと VNet 統合を組み合わせることで、API Management を送受信共に閉域化（プライベート IP アドレスのみでの通信）することができるようになりました。

![alt text](/images/private-apim/private-architecture.png)

今までは API Management で送受信共に閉域化するには、Premium プランを使った VNet インジェクションを行う必要がありましたが、Standard v2 プランであれば、VNet インジェクションを行わずに、プライベート エンドポイントを使用して閉域化することができるようになりました。

これによって以下のように料金を抑えることができるようになります。

| プラン      | 料金           |
| ----------- | -------------- |
| Standard v2 | \99,999/月     |
| Premium     | \399,304.02/月 |

参考：[API Management の価格](https://azure.microsoft.com/ja-jp/pricing/details/api-management/?msockid=0679d68acdec65fa3197c27ecc946464)

> Premium v2 プランは、まだプライベート エンドポイントに対応していないので VNet インジェクションを行うことになります。
>
> 参考：[プライベート仮想ネットワークに Azure API Management インスタンスを挿入する - Premium v2 レベル](https://learn.microsoft.com/ja-jp/azure/api-management/inject-vnet-v2)

約 30 万円安くなるのは大きいですね！

今回は、Standard v2 プランの API Management を閉域化する手順を紹介します。

# 構成

以下の構成で API Management を閉域化していきます。

- API Management
- Azure Functions（バックエンド API）
- VPN Gateway（ローカル PC から閉域環境に入って API Management にアクセスするため）

![alt text](/images/private-apim/architecture.png)

# リソース グループの作成

> APIM 作成フローの中で新規リソース グループを作成しようとすると、さらにその中でプライベート エンドポイント作成フローを展開したときに、既存のリソース グループからしか選択できないという不具合があるので、あらかじめリソース グループを作成しておく必要があります。

![alt text](/images/private-apim/create-rg-1.png)

![alt text](/images/private-apim/create-rg-2.png)

# 仮想ネットワークの作成

> リソース グループと同様に APIM 作成フローの中で柔軟に仮想ネットワークやサブネットを作成することができないので、あらかじめ作成しておきます。

![alt text](/images/private-apim/create-vnet-1.png)

![alt text](/images/private-apim/create-vnet-2.png)

仮想ネットワーク内に以下の 2 つのサブネットを作成します。

- `apim-subnet`：API Management の VNet 統合用サブネット
- `pe-subnet`：プライベート エンドポイント用サブネット

![alt text](/images/private-apim/create-vnet-3.png)

<!-- ![alt text](/images/private-apim/create-vnet-4.png) -->

# API Management の作成

次に API Management を作成します。

![alt text](/images/private-apim/create-apim-1.png)

価格レベルは Standard v2 を選択します。

![alt text](/images/private-apim/create-apim-2.png)

「仮想ネットワーク」タブで「Inbound private link and/or outbound virtual network integration」を選択し閉域化の設定をしていきます。

![alt text](/images/private-apim/create-apim-3.png)

「Inbound features」セクションにて「+ 新規作成」を選択し、プライベート エンドポイントを作成します。配置するサブネットはプライベート エンドポイント用サブネットを選択するようにします。

また、プライベート DNS 統合を選択することで、API Management のプライベート エンドポイントの DNS 名を自動的に解決できるようになりますので、必ず選択しておきましょう。

![alt text](/images/private-apim/create-apim-4.png)

プライベート エンドポイントが作成出来たら、「Public network accesss」を「Disable」に変更し、「Outbound features」セクションにて、「Virtual network integration」を「Enable」にして VNet 統合用のサブネットを選択することで、API Management の VNet 統合を行います。

![alt text](/images/private-apim/create-apim-5.png)

そのまま「確認と作成」を選択し、API Management を作成します。

# Azure Functions の作成

次にバックエンド API として Azure Functions を作成します。今回は「[クイックスタート: コマンド ラインから Azure に Python 関数を作成する](https://learn.microsoft.com/ja-jp/azure/azure-functions/create-first-function-cli-python?tabs=windows%2Cpowershell%2Cazure-cli%2Cbrowser)」を参考に Python 3.12 を使用した HTTP トリガーの Azure Functions を作成します。お好きな言語やツールを使って関数を作成してもらっても構いません。

ホスティング プランはなんでもいいのですが、せっかくなので今回は、最近東日本リージョンでも提供が開始されたフレックス従量課金プランを選択します。

![alt text](/images/private-apim/create-function-1.png)

![alt text](/images/private-apim/create-function-2.png)

![alt text](/images/private-apim/create-function-3.png)

今回は閉域化された API を作成するのでパブリック アクセスを無効にします。

![alt text](/images/private-apim/create-function-4.png)

Application Insights は今回は不要なので無効にします。

![alt text](/images/private-apim/create-function-5.png)

Azure Functions からストレージへのリソース認証にはマネージド ID を使用します。これによって自動的に推奨される役割（ロール）が付与ユーザー割り当てマネージド ID が作成され Azure Functions に割り当てられます。

![alt text](/images/private-apim/create-function-6.png)

## Azure Functions 用 Private Endpoint の作成

API Management と同様に Azure Functions もプライベート エンドポイントを作成します。これによって VNet 統合された API Management から Azure Functions にアクセスできるようになります。

Azure Functions リソースの「ネットワーク」ブレードから「プライベート エンドポイント」を選択します。

![alt text](/images/private-apim/create-pe-for-function-1.png)

「追加」⇒「簡易」からプライベート エンドポイントを作成します。

![alt text](/images/private-apim/create-pe-for-function-2.png)

プライベート エンドポイント用のサブネットを選択し、プライベート DNS ゾーン統合のオプションも選択した上で「OK」を選択します。

![alt text](/images/private-apim/create-pe-for-function-3.png)

ここまでで以下が完了しました。

- 送受信が閉域化された API Management
- 受信が閉域化された Azure Functions

# VPN Gateway の作成

閉域化された API Management にクライアントとしてローカル PC からアクセスするために、VPN Gateway を作成していきます。

以下の構成で VPN Gateway を作成します。VPN Gateway のデプロイには非常に時間がかかるので、お風呂に入っている間に作成しておくと良いでしょう。

![alt text](/images/private-apim/create-vpngw-1.png)

VPN クライアント プロファイルを作成してダウンロードしていきます。

「ポイント対サイトの構成」ブレードで以下の設定を行います。

- トンネルの種類：OpenVPN (SSL)
- 認証の種類：Azure Active Directory (Entra ID)

Azure Active Directory (Entra ID) の情報は、[こちら](https://learn.microsoft.com/ja-jp/azure/vpn-gateway/point-to-site-entra-gateway#configure-vpn) のドキュメントを参考にテナント ID を使って設定してください。

![alt text](/images/private-apim/create-vpngw-2.png)

設定を保存したら、「VPN クライアントのダウンロード」をクリックして VPN クライアント プロファイル構成パッケージを取得します。

ダウンロードした構成パッケージを確認します。 ダウンロードした zip ファイルを解凍し、`AzureVPN` フォルダ内に "azurevpnconfig.xml" が存在することを確認します。

## VPN Gateway の接続

> 前提：Azure VPN クライアントがインストールされていない場合は、[こちら](https://learn.microsoft.com/ja-jp/azure/vpn-gateway/point-to-site-entra-vpn-client-windows#download) のドキュメントを参考にインストールしてください。

Azure VPN クライアントを起動し、左下の「+」をクリックして「インポート」を選択します。

![alt text](/images/private-apim/connect-vpn-1.png)

先ほどダウンロードした "azurevpnconfig.xml" を開き構成をインポートします。

任意の接続名を入力し、「保存」をクリックします。

![alt text](/images/private-apim/connect-vpn-2.png)

接続名のプロファイルに対して「接続」をクリックして接続を開始します。

![alt text](/images/private-apim/connect-vpn-3.png)

ターミナルやコマンド プロンプトで接続を確認します。

```powershell
ipconfig
```

![alt text](/images/private-apim/check-vpn-connection.png)

## hosts の設定

VPN Gateway に接続したら、API Management のプライベート エンドポイントの DNS 名を解決できるように hosts を設定します。

```powershell
$ResourceGroup = "<RG_NAME>"

$zones = az network private-dns zone list --resource-group $ResourceGroup --query '[].name' -o tsv

foreach ($zone in $zones) {
    az network private-dns record-set a list --resource-group $ResourceGroup --zone-name $zone --query '[].{IP: aRecords[0].ipv4Address, FQDN: fqdn}' -o tsv |
    ForEach-Object { $_ -replace 'privatelink\.', '' -replace 'vaultcore', 'vault' -replace '\.$', '' }
}

# 出力例
10.0.1.4        private-apim-sample.azure-api.net
10.0.1.5        sample-private-api.azurewebsites.net
10.0.1.5        sample-private-api.scm.azurewebsites.net
```

出力されたプライベート IP アドレスとドメイン名のペアをコピーし、hosts ファイルに追加します。

> Windows 11 の場合、hosts ファイルは C:\Windows\System32\drivers\etc\hosts にあります。

これでローカル PC から API Management のプライベート エンドポイントの DNS 名を解決できるようになります。

## API のデプロイ

上記の手順でローカル PC から Azure Functions にもアクセスできるようにもなっているので下記の手順で API をデプロイします。

> Azure Functions Core Tools をインストールしていない場合は、[こちら](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-run-local?tabs=windows%2Cisolated-process%2Cnode-v4%2Cpython-v2%2Chttp-trigger%2Ccontainer-apps&pivots=programming-language-python#install-the-azure-functions-core-tools) のドキュメントを参考にインストールしてください。

```powershell
mkdir HttpExample
cd HttpExample

python -m venv .venv
.venv\scripts\activate

func init --python
func new --name HttpExample --template "HTTP trigger" --authlevel "anonymous"
func azure functionapp publish <APP_NAME>
```

これで Azure Functions に API をデプロイすることができました。

# API Management へ API の追加

API Management の「API」ブレードから「Function App」を選択し、Azure Functions を選択します。

![alt text](/images/private-apim/add-api-1.png)

![alt text](/images/private-apim/add-api-2.png)

「サブスクリプション」ブレードからサブスクリプション キーを確認し、手元にコピーしておきます。

![alt text](/images/private-apim/copy-subscription-key.png)

以下のコマンドで API Management を通して Azure Functions にアクセスできることを確認します。

```powershell
$apiUrl = "https://<your-apim-name>.azure-api.net/<api-path>"
$subscriptionKey = "<your-subscription-key>"

$response = Invoke-WebRequest -Uri $apiUrl -Method 'POST' -Headers @{
    "Ocp-Apim-Subscription-Key" = $subscriptionKey
}

$response.Content
```

また、VPN 接続を切断した状態で、API Management のプライベート エンドポイントの DNS 名を解決できないことを確認します。

```powershell
$response = Invoke-WebRequest -Uri $apiUrl -Method 'POST' -Headers @{
    "Ocp-Apim-Subscription-Key" = $subscriptionKey
}

# 出力例
Invoke-WebRequest: 接続済みの呼び出し先が一定の時間を過ぎても正しく応答しなかったため、接続できませんでした。または接続済みのホストが応答しなかったため、確立された接続は失敗しました。
```

これによって API Management が送受信共に閉域化されていることを確認することができました。

# まとめ

API Management の Standard v2 プランにおいて、プライベート エンドポイントのパブリック プレビューが開始されたことで、API Management を安価に閉域化することができるようになりました。これによって、API Management を使用した API の閉域化が非常に簡単になりました。

また、API Management の VNet 統合を使用することで、API Management から Azure Functions などのバックエンド API にも閉域化された状態でアクセスできるようになります。

これによって完全に閉域化された環境で、API Management を使用して API の管理を行うことができるようになります。

# 参考資料

- [Quickstart: Create a new Azure API Management instance by using the Azure portal](https://learn.microsoft.com/ja-jp/azure/api-management/get-started-create-service-instance)
- [送信接続のためにプライベート仮想ネットワークと Azure API Management インスタンスを統合する](https://learn.microsoft.com/ja-jp/azure/api-management/integrate-vnet-outbound)
- [受信プライベート エンドポイントを使用して API Management に非公開で接続する](https://learn.microsoft.com/ja-jp/azure/api-management/private-endpoint?tabs=v2)
- ["Minimum Viable Sample" 機能最小限な社内文書 RAG チャットボット(プライベートネットワーク縛り)](https://github.com/torumakabe/rag-chat-private-minimal)
