---
title: "ローカル PC から Azure 閉域環境へ接続する方法【VPN Gateway 入門】"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure", "vpn", "network"]
published: true
publication_name: "microsoft"
---

Azure 上に構築したリソースを、**完全に閉域化**（プライベートネットワーク化）したいという要件はよくあります。たとえば以下のようなケースです。

-   Azure Functions や App Service をパブリックから遮断し、プライベート エンドポイント経由でのみアクセス可能にしたい
-   Web アプリを VNet 統合し、外部からのアクセスを制限したい
-   API Management を内部ネットワーク向けに構成し、インターネット経由では利用できないようにしたい

このような「閉じた環境」においても、ローカル PC から開発やテストのために一時的にアクセスしたい場面は多くあります。

この記事では、**Azure VPN Gateway を使ってローカル PC を Azure の仮想ネットワークに接続する手順**を丁寧に解説します。加えて、**プライベート DNS 名の名前解決**をローカルでも行えるようにする方法まで紹介しています。

図やスクリプトを交えて解説しますので、初めて VPN Gateway を扱う方にもわかりやすい内容になっていると思います。

# VPN Gateway の作成

閉域化された Azure リソース にクライアントとしてローカル PC からアクセスするために、VPN Gateway を作成していきます。

以下の構成で VPN Gateway を作成します。VPN Gateway のデプロイには非常に時間がかかるので、お風呂に入っている間に作成しておくと良いでしょう。

![alt text](/images/private-apim/create-vpngw-1.png)

> 作成時は仮想ネットワークとサブネットのアドレス範囲を指定します。そうすると、VPN Gateway 用のサブネットが自動的に作成されます。

# VPN Gateway の構成

VPN クライアント プロファイルを作成してダウンロードしていきます。

「ポイント対サイトの構成」ブレードで以下の設定を行います。

-   **トンネルの種類：OpenVPN (SSL)**
-   **認証の種類：Azure Active Directory (Entra ID)**

Azure Active Directory (Entra ID) の情報は、[こちら](https://learn.microsoft.com/ja-jp/azure/vpn-gateway/point-to-site-entra-gateway#configure-vpn) のドキュメントを参考にテナント ID を使って設定してください。

![alt text](/images/private-apim/create-vpngw-2.png)

設定を保存したら、「**VPN クライアントのダウンロード**」をクリックして VPN クライアント プロファイル構成パッケージを取得します。

ダウンロードした構成パッケージを確認します。 ダウンロードした zip ファイルを解凍し、`AzureVPN` フォルダ内に **"azurevpnconfig.xml"** が存在することを確認します。

# VPN Gateway の接続

> 前提：Azure VPN クライアントがインストールされていない場合は、[こちら](https://learn.microsoft.com/ja-jp/azure/vpn-gateway/point-to-site-entra-vpn-client-windows#download) のドキュメントを参考にインストールしてください。

Azure VPN クライアントを起動し、左下の「+」をクリックして「インポート」を選択します。

![alt text](/images/private-apim/connect-vpn-1.png)

先ほどダウンロードした **"azurevpnconfig.xml"** を開き構成をインポートします。

任意の接続名を入力し、「保存」をクリックします。

![alt text](/images/private-apim/connect-vpn-2.png)

接続名のプロファイルに対して「接続」をクリックして接続を開始します。

![alt text](/images/private-apim/connect-vpn-3.png)

ターミナルやコマンド プロンプトで接続を確認します。

```powershell
ipconfig
```

![alt text](/images/private-apim/check-vpn-connection.png)

# hosts の設定

VPN Gateway に接続したら、Azure リソースのプライベート エンドポイントの DNS 名を解決できるように hosts を設定します。

以下のスクリプトを実行することで、指定したリソース グループにある **すべての Azure プライベート DNS ゾーン内の A レコード** を一覧表示し、見やすく整形して出力します。アクセスしたいリソースがあるリソース グループを指定してください。

> 今回のスクリプトは、Microsoft 真壁さんの「"Minimum Viable Sample" 機能最小限な社内文書 RAG チャットボット(プライベートネットワーク縛り) 」リポジトリ内のスクリプトを流用させていただいております。
>
> 参考：["Minimum Viable Sample" 機能最小限な社内文書 RAG チャットボット(プライベートネットワーク縛り)](https://github.com/torumakabe/rag-chat-private-minimal/tree/main/scripts/util/hosts)

### PowerShell

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

### Bash

```bash
#!/usr/bin/env bash

set -o pipefail

if [ $# -eq 0 ]; then
  echo "Usage: $0 <resource_group>"
  exit 1
fi

zones=$(az network private-dns zone list --resource-group "$1" --query '[].name' -o tsv)

for zone in $zones; do
  az network private-dns record-set a list --resource-group "$1" --zone-name "$zone" --query '[].{IP: aRecords[0].ipv4Address, FQDN: fqdn}' -o tsv  | sed 's/privatelink\.//' | sed 's/vaultcore/vault/' | sed 's/\.$//'
done

# 出力例
10.0.1.4        private-apim-sample.azure-api.net
10.0.1.5        sample-private-api.azurewebsites.net
10.0.1.5        sample-private-api.scm.azurewebsites.net
```

出力されたプライベート IP アドレスとドメイン名のペアをコピーし、`hosts` ファイルに追加します。

> Windows 11 の場合、hosts ファイルは C:\Windows\System32\drivers\etc\hosts にあります。

# まとめ

この記事では、ローカル PC から Azure の閉域ネットワークに安全にアクセスするための手順を紹介しました。

-   **VPN Gateway の作成**と**ポイント対サイト接続の構成**
-   Azure VPN クライアントを使ったローカル PC からの接続方法
-   **プライベート DNS 名の名前解決**に必要な `hosts` ファイルの整備

これらのステップを通じて、**APIM や Azure Functions などの閉域リソースに対して、ローカルからのアクセスやデプロイが可能**になります。

特に開発・検証フェーズでは、ローカルからの疎通確認ができることは大きなメリットです。一方で、本番環境では不要なアクセス許可を残さないよう、VPN 接続の管理には十分注意してください。
