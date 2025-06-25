---
title: "既存 API サーバーを MCP サーバーに一瞬で変える方法"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# はじめに

この度、Azure API Management (APIM) で、既存の API を MCP サーバーとして公開する機能がプレビューでリリースされました。

猫も杓子も MCP な時代ですが、既存の API サーバーを MCP サーバーに変えるのは中々大変です。企業人であれば、コーディング以外の大変さもあるでしょう。

![手動でコーディングして MCP 化するのと APIM の機能を使うことの比較図](/images/expose-mcp-server-in-apim/comparison.png)

今後は、APIM にインポートされている API であれば MCP の皮を被せて MCP サーバーとして公開することができるようになります。これにより以下のような恩恵を受けることができます。

-   **MCP サーバーの構築が簡単**: 既存の API を APIM にインポートするだけで MCP サーバーとして公開できます。
-   **APIM の通常恩恵**: APIM の通常の機能（認証、レート制限、ログ収集など）をそのまま利用できます。
    -   **認証**: API へのアクセス制御を簡単に設定できます（`例：社内かつ特定の社員しかアクセスさせない`）。
    -   **レート制限**: API の使用量を制限し、過負荷を防ぐことができます（`例：1 分あたりのリクエスト数上限を設ける`）。
    -   **ログ収集**: API の使用状況を詳細に把握できます（`誰がどのツールをどれくらい使用しているか`）。

APIM を使った API のキーレス認証の設定方法については、こちらの記事で詳しく解説していますので、ぜひご覧ください。

https://zenn.dev/microsoft/articles/apim-backend-api-auth-using-managed-id

また、APIM（API 群を含めて）を閉域化する方法については以下の記事で解説しています。

https://zenn.dev/microsoft/articles/private-apim

# API Management を作成

まずは、APIM リソース一覧ページに移動し、上部の「+ 作成」を選択します。

![Azure ポータルの APIM リソース一覧ページ](/images/expose-mcp-server-in-apim/create-apim-1.png)

基本タブにて以下のように入力します。

-   **リソース グループ**: 既存のリソース グループを選択するか、新規作成します。
-   **リージョン**: APIM を配置するリージョンを選択します。例: `Japan East`
-   **リソース名**: APIM の名前を入力します。例: `mcp-apim`
-   **組織名**: APIM の組織名を入力します。例: `Contoso`
-   **管理者のメール アドレス**: APIM の管理者のメールアドレスを入力します。
-   **価格レベル**: `Basic` を選択します。

![Azure ポータルの APIM リソース作成ページ](/images/expose-mcp-server-in-apim/create-apim-2.png)

> 現在、この「API を MCP サーバーとして公開する機能」は、**`Basic` と`Standard` と `Premium` の価格レベルでのみ** 利用可能です。
>
> 参考：[API Management で MCP サーバーとして REST API を公開する](https://learn.microsoft.com/ja-jp/azure/api-management/export-rest-mcp-server)

他の設定項目はデフォルトのままにしておき、下部の「確認と作成」を選択し、「作成」から APIM の作成を開始します。

APIM のデプロイは少し時間がかかります（体感 10 分程度）。デプロイが完了するまで待ちます。

# APIM の更新グループを編集

現在、この「API を MCP サーバーとして公開する機能」は、**AI ゲートウェイ先行アクセス** グループに対してリリースされています。APIM はデフォルトで **既定** グループに属しているため、まずは APIM の更新グループを編集します。

作成した APIM リソース ページの「サービスの更新（プレビュー）」ブレードに移動して、更新グループの編集を行います。

![Azure ポータルの APIM 更新グループ編集ページ](/images/expose-mcp-server-in-apim/update-apim-1.png)

更新グループの一覧から「AI ゲートウェイ先行アクセス」を選択し、下部の「保存」を選択します。

![Azure ポータルの APIM 更新グループ編集ページ](/images/expose-mcp-server-in-apim/update-apim-2.png)

[公式ドキュメント](https://learn.microsoft.com/ja-jp/azure/api-management/export-rest-mcp-server) によると「**グループに参加した後、MCP サーバーの機能にアクセスするには 2 時間かかることがあります。**」とのことですので、しばらく待ちます。

しばらくすると、APIM リソースに「MCP」ブレードが追加されます。

![Azure ポータルの APIM MCP ブレード](/images/expose-mcp-server-in-apim/mcp-blade.png)

# API をインポートし発行する

次に APIM に MCP サーバーとして公開したい API をインポートします。

今回は、[Petstore API](https://petstore3.swagger.io/) を使います。実際には、既存の社内 API や、それをホストしている Azure Functions などのリソースをインポートすることができます。

# MCP サーバーとして公開

# GitHub Copilot (VS Code) から使ってみる

# まとめ
