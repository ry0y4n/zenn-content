---
title: "【5分で理解】Azure AI Agent Service を速攻で理解しよう"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [azure, ai, aiエージェント]
published: true
publication_name: "microsoft"
---

こんにちは、モモスケです。

今回は、**Azure AI Agent Service** の概要とその主要な機能について、**初心者向けに分かりやすく解説**します。

**具体的なコードは省略しつつ、サービスの全体像や注目すべきポイントをピックアップしてご紹介します。**

::: message
実際にエージェントを動かしてみたい人は、**[クイック スタート: 新しいエージェントを作成する](https://learn.microsoft.com/ja-jp/azure/ai-services/agents/quickstart?view=azure-python-preview&pivots=programming-language-python-azure)** をご覧ください。

また、以下のような他の方の記事も参考にしてみてください。

-   [Azure AI Agent Service で簡単 RAG を実装する](https://zenn.dev/microsoft/articles/aiagentservice-dotnet-01)
-   [Bing API を利用した Azure AI Agent Service の実装ログ](https://zenn.dev/dddsss/articles/3237af8dd463fa)
-   [Azure AI Agent を動かしてエージェンティックワールドに入門する話](https://zenn.dev/headwaters/articles/3a47816751441b)

:::

# Azure AI Agent Service とは？

https://learn.microsoft.com/ja-jp/azure/ai-services/agents/overview?view=azure-python-preview

Azure AI Agent Service は、基盤となるコンピューティングリソースやストレージの管理を意識することなく、高品質でスケーラブルな AI エージェントを安全に構築・デプロイできる**フルマネージドサービス**です。  
開発者は、[**AI Foundry ポータル**](https://learn.microsoft.com/en-us/azure/ai-studio/what-is-ai-studio)、[**OpenAI SDK**](https://learn.microsoft.com/ja-jp/azure/ai-services/agents/quickstart?pivots=programming-language-python-openai)、[**Azure AI Foundry SDK**](https://learn.microsoft.com/ja-jp/azure/ai-services/agents/quickstart?programming-language-python-azure) などのツールを活用することで、短時間でエージェントを構築できる点が大きな魅力です。

[AI Agent Service の主なクォータと制限](https://learn.microsoft.com/ja-jp/azure/ai-services/agents/quotas-limits?view=azure-python-preview#quotas-and-limits-for-the-azure-ai-agent-service)は以下の通りです。

| 制限名                                                         | 制限値                                                                                                        |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| エージェント/スレッドあたりの最大ファイル数                    | API または Azure AI Foundry ポータルを使用する場合は 10,000 個。 Azure OpenAI Studio では、制限は 20 でした。 |
| エージェントの最大ファイル サイズと微調整                      | 512 MB                                                                                                        |
| エージェント用にアップロードされたすべてのファイルの最大サイズ | 100 GB                                                                                                        |
| エージェント トークンの制限                                    | 2,000,000 トークンの制限                                                                                      |

# AI Agent vs. Azure OpenAI Assistant

[Azure OpenAI Assistants](https://learn.microsoft.com/ja-jp/azure/ai-services/openai/concepts/assistants) を使用してもファイル取得や Code Interpreter を搭載したエージェント構築が可能ですが、Azure AI Agent Service は以下の点で優れています。

-   **幅広いモデル選択肢**  
    OpenAI のモデル以外にも、Llama 3、Mistral、Cohere など、多様な AI モデルを選択可能。

-   **データ結合の柔軟性**  
     Bing 検索、AI Search、Azure Functions やその他 API とのデータ結合が容易で、エージェントの応答にデータ ソースを柔軟に組み込むことができます。

-   **セキュリティと柔軟なストレージ**  
    保護されたデータ処理、キーレス認証、パブリックエグレスの無効化などの高度なセキュリティ機能に加え、ストレージに Blob Storage や Microsoft マネージドストレージを選択できる自由度があります。

> エンタープライズ要件がある場合は、Azure AI Agent Service の使用を検討することをお勧めします。（引用元：[Azure エージェントと Azure OpenAI アシスタントとの比較](https://learn.microsoft.com/ja-jp/azure/ai-services/agents/overview?view=azure-python-preview#comparing-azure-agents-and-azure-openai-assistants)）

---

# 多彩なツール群で実現する高度な機能

Azure AI Agent Service は、**ナレッジ ツール**と**アクション ツール**という 2 種類のツールを提供し、エージェントのパフォーマンスと利便性を飛躍的に向上させます。

## ナレッジ ツール

### [**Bing 検索によるグラウンディング**](https://learn.microsoft.com/ja-jp/azure/ai-services/agents/how-to/tools/bing-grounding?view=azure-python-preview&tabs=python&pivots=overview)

エージェントが応答を生成する際に、リアルタイムでパブリック Web から最新情報を取得し回答に組み込むことが出来ます。

このツールを有効にしていると、ユーザーがクエリを送信したとき、AI Agent は Bing 検索を使用したグランディングを活用するかどうかを自動で判断します。

-   クォータ：**150 トランザクション/秒、1M トランザクション/日**
-   料金：**1,000 トランザクションあたり $35**

最終的にエージェントからの応答には、生成するために使用された **Web サイトへのリンク**と、検索に使用された **Bing クエリへのリンク**が含まれます。

### [**ファイル検索**](https://learn.microsoft.com/ja-jp/azure/ai-services/agents/how-to/tools/file-search?view=azure-python-preview&tabs=python&pivots=azure-blob-storage-code-examples)

エージェントに対してファイルを検索する機能を提供します。PPTX、DOCX などの多くのファイル形式に対応し、アップロードされたファイルは **自動解析、チャンキング、埋め込み処理** を経て **ベクトルデータベース** に格納されます。

::: message
[サポートされているファイルの種類](https://learn.microsoft.com/ja-jp/azure/ai-services/agents/how-to/tools/file-search?tabs=python&pivots=supported-filetypes)
:::

ファイル検索ツールは、以下の 2 つのソースからファイルを取得できます。

-   **ローカルファイルのアップロード**
-   **Azure Blob Storage**

#### **エージェントのセットアップ方法**

ファイル検索ツールには **Basic** と **Standard** の 2 種類のセットアップ方法があります。

**Basic エージェントのセットアップ**

-   アップロードされたファイルは **Microsoft マネージドのストレージ** に保存される。
-   **Microsoft マネージドの検索リソース** を使用して **ベクトルストア** が作成される。

![alt text](/images/introduce-ai-agent-summary/basic-agent.png)

**Standard エージェントのセットアップ**

-   接続された **Blob Storage** リソースを活用し、ファイルを保存。
-   接続された **AI Search リソース** を使用してベクトルストアを作成。

![alt text](/images/introduce-ai-agent-summary/standard-agent.png)

どちらのセットアップでも Azure OpenAI が以下のプロセスを処理します。

-   **ドキュメントの自動解析とチャンキング**
-   **埋め込みの作成と保存**
-   **ユーザーのクエリに対してベクトル検索とキーワード検索の両方を実行**

::: message
**コードの違いはなく**、唯一の違いは **ファイルとベクトルストアの保存場所** です。
:::

#### **検索の仕組み**

ファイル検索ツールは、以下の最適化を行いながら、ユーザーのクエリに基づいて検索を実行します。

✅ **検索の最適化**

-   **ユーザーのクエリを書き換え**、検索用に最適化
-   **複雑なクエリを複数の検索に分割**し、並列で実行
-   **キーワードとベクトルのハイブリッド検索 + セマンティック検索** を実行

✅ **既定の設定**

-   **チャンクサイズ**：800 トークン
-   **チャンクオーバーラップ**：400 トークン
-   **埋め込みモデル**：256 次元の `text-embedding-3-large`
-   **コンテキストに追加される最大チャンク数**：20

#### **制約**

-   **最大ファイルサイズ**：512 MB
-   **ファイルごとの最大トークン数**：5,000,000 トークン以下（添付時に自動計算）

::: message
既存の AI Search インデックスを AI Agent に接続することも可能です。  
[詳細はこちら](https://learn.microsoft.com/ja-jp/azure/ai-services/agents/how-to/tools/azure-ai-search?view=azure-python-preview&tabs=azurecli%2Cpython&pivots=overview-azure-ai-search)
:::

## アクションツール

外部サービスやコード実行の柔軟な統合を実現するために、**アクションツール** を提供しています。以下のツールを活用することで、より高度な AI エージェントを構築できます。

---

### [**Function Calling**](https://learn.microsoft.com/ja-jp/azure/ai-services/agents/how-to/tools/function-calling?tabs=python&pivots=code-example)

エージェントが呼び出せる外部の関数を定義することができます。

-   **AI Agent SDK** によってサポートされている。
-   関数の実行は、作成後 **最大 10 分間** の制限があり、それまでにツールの出力を送信する必要がある。

#### **エージェント作成時のサンプルコード**

```python
import json
import datetime
from typing import Any, Callable, Set, Dict, List, Optional
import os
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from azure.ai.projects.models import FunctionTool, ToolSet
from user_functions import user_functions

# 呼び出したいユーザー定義の関数
def fetch_weather(location: str) -> str:
    """
    Fetches the weather information for the specified location.

    :param location (str): The location to fetch weather for.
    :return: Weather information as a JSON string.
    :rtype: str
    """
    # モックの天気データ
    mock_weather_data = {"New York": "Sunny, 25°C", "London": "Cloudy, 18°C", "Tokyo": "Rainy, 22°C"}
    weather = mock_weather_data.get(location, "Weather data not available for this location.")
    weather_json = json.dumps({"weather": weather})
    return weather_json

# ユーザー定義の関数をセットに追加
user_functions: Set[Callable[..., Any]] = {
    fetch_weather,
}

# Azure AI Foundry からコピーした接続文字列を使用して Azure AI クライアントを作成
project_client = AIProjectClient.from_connection_string(
    credential=DefaultAzureCredential(),
    conn_str=os.environ["PROJECT_CONNECTION_STRING"],
)

# ユーザー定義の関数を関数ツールに追加
functions = FunctionTool(user_functions)
toolset = ToolSet()
toolset.add(functions)

# Function Calling のツールを指定しつつ、エージェントを作成
agent = project_client.agents.create_agent(
    model="gpt-4o-mini",
    name="my-agent",
    instructions="You are a helpful agent",
    toolset=toolset
)
print(f"Created agent, ID: {agent.id}")
```

### [**Code Interpreter**](https://learn.microsoft.com/ja-jp/azure/ai-services/agents/how-to/tools/code-interpreter?tabs=python&pivots=overview)

Code Interpreter は、エージェントが サンドボックス環境で Python コードを実行 できるツールです。データ解析やカスタム計算処理をエージェント内で完結させることが可能です。

-   エージェントは実行に失敗すると、コードを自動的に修正・再実行できる。
-   AOAI のトークン料金とは別に、追加料金が発生する：
    -   **$0.03/セッション**
    -   既定ではセッションは **1 時間アクティブ**（セッションの間は追加料金なし）
    -   2 つのスレッドで同時実行すると 2 セッション分の料金が発生

::: message
つまり、同じスレッド内で 1 時間まで連続して指示を出しても、課金は 1 セッション分のみ。
:::

### [**OpenAPI Specified Tools**](https://learn.microsoft.com/ja-jp/azure/ai-services/agents/how-to/tools/openapi-spec?tabs=python&pivots=overview)

このツールを活用すると、**AI Agent を外部 API に接続** できるようになります。
これにより、さまざまなアプリケーションとのスケーラブルな相互運用性を実現できます。

-   OpenAPI 3.0 仕様に基づいた HTTP API の記述が可能。
-   マネージド ID による認証 をサポートし、セキュリティ強化が可能。

サポートされる認証方法

-   Anonymous（匿名）
-   API Key
-   マネージド ID

::: message
OpenAPI Specification の詳細はこちら: [OpenAPI Specification v3.1.1](https://spec.openapis.org/oas/latest.html)
:::

### [**Azure Functions**](https://learn.microsoft.com/ja-jp/azure/ai-services/agents/how-to/tools/azure-functions?pivots=overview)

Azure Functions は、**イベント駆動型の関数実行** を提供するツールで、AI Agent が外部のシステムやサービスとやり取りするのに最適です。

-   **関数のトリガーとバインド** を活用することで、エージェントの機能を拡張可能。
-   **イベントに応じた処理を実行し、AI Agent とスムーズに連携できる。**

**活用シナリオ**
顧客メッセージをチャットボットが受信するたびに Azure Functions をトリガーし、
AI Agent を通じてリアルタイム応答を送信する 仕組みを構築できます。

::: message
公式サンプルリポジトリ: [Azure-Samples/azure-functions-ai-services-agent-python](https://github.com/Azure-Samples/azure-functions-ai-services-agent-python)
:::

# コンテンツ フィルタリング

Azure AI Agent Service は、**コンテンツ フィルタリング**機能を内蔵しており、**暴力、憎悪、性的表現、自傷行為** の 4 つカテゴリーをそれぞれ 4 つの重大度で自動検出します（デフォルトでは「**中**」のフィルタが適用され、必要に応じて設定変更が可能です）。

コンテンツ フィルタリングを部分的または完全にオフにするには、[専用フォーム](https://ncv.microsoft.com/uEfCgnITdR)から申請して承認される必要があります。

既定の 4 つのカテゴリに加えて、以下のフィルター カテゴリも構成可能です。

| フィルター カテゴリー                                  | 状態       | 既定の設定 | プロンプトと入力候補のどちらに適用されますか? | 説明                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| ------------------------------------------------------ | ---------- | ---------- | --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 直接攻撃に関するプロンプト シールド (ジェイルブレイク) | GA         | オン       | ユーザー プロンプト                           | ジェイルブレイク リスクがあるかもしれないユーザー プロンプトをフィルター処理/注釈付けします。 注釈の詳細については、「[Azure AI Foundry のコンテンツ フィルタリング](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/content-filter?tabs=python#annotations-preview)」を参照してください。                                                                                                                                    |
| 間接攻撃に関するプロンプト シールド                    | GA         | オフ       | ユーザー プロンプト                           | 生成 AI システムがアクセスして処理できるドキュメント内に、第三者が悪意のある命令を配置する潜在的な脆弱性である間接攻撃 (別名、間接プロンプト攻撃またはクロスドメイン プロンプト インジェクション攻撃) をフィルター処理/注釈付けします。 必須: [ドキュメントの埋め込みと書式設定](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/content-filter?tabs=warning%2Cuser-prompt%2Cpython-new#embedding-documents-in-your-prompt)。 |
| 保護された素材 - コード                                | GA         | オン       | Completion                                    | 保護されたコードをフィルター処理するか、GitHub Copilot を利用して何らかのパブリック コード ソースと一致するコード スニペット用の注釈内の引用とライセンスの情報の例を取得します。 注釈の使用に関する詳細については、「[コンテンツのフィルター処理の概念のガイド](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/content-filter#annotations-preview)」を参照してください                                                       |
| 保護された素材 - テキスト                              | GA         | オン       | Completion                                    | 既知のテキスト コンテンツを識別し、モデル出力内でそれが表示されることをブロックします (たとえば、曲の歌詞、レシピ、選択した Web コンテンツなど)。                                                                                                                                                                                                                                                                                                 |
| グラウンデッドネス                                     | プレビュー | オフ       | Completion                                    | 大規模言語モデル (LLM) のテキスト応答が、ユーザーが提供するソース資料に基づいているかどうかを検出します。 根拠なしとは、ソース資料に存在していた事実に基づかない、または不正確な情報が LLM から生成されることを指します。 必須: [ドキュメントの埋め込みと書式設定](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/content-filter?tabs=warning%2Cuser-prompt%2Cpython-new#embedding-documents-in-your-prompt)。               |

引用元：[その他のフィルターについて](https://learn.microsoft.com/ja-jp/azure/ai-services/openai/how-to/content-filters?context=%2Fazure%2Fai-services%2Fagents%2Fcontext%2Fcontext#understand-other-filters)

# データの取り扱い

また、エージェントとのやり取りで生成されるデータは、以下のように安全かつ効率的に管理されます。

-   **データの利用**：トレーニング目的では使用されず、アップロードされたファイルやメッセージは専用のストレージに安全に保管されます。
-   **格納場所**：Microsoft が管理するセキュアなストレージアカウントに保存され、地理的には、**AI Agent Service エンドポイントと同一のリージョン**で保存。
-   **保持期間**：明示的に削除するまで保持され、SDK を利用して簡単に管理・削除が可能です。

# 今後の展開と未来の可能性

最新の情報（2025 年 2 月 13 日現在）によると、Azure AI Agent Service は以下の新機能の展開を予定しています。

-   **プライベート仮想ネットワーク**
    より強固なセキュリティ環境の提供に向けた取り組み。

-   **スレッドストレージ**
    エージェントのやり取りデータの管理がさらに効率的になる機能の実装。

これらの機能追加により、さらに多様なユースケースに対応可能となり、次世代の AI オートメーションが加速することが期待されます。

# まとめ

Azure AI Agent Service は、開発者や企業が手軽に高度な AI エージェントを構築・デプロイできる画期的なプラットフォームです。

-   **多彩なツール群**が現実の業務フローにシームレスに統合できる
-   **セキュリティとパフォーマンス**を両立し、エンタープライズ要件にも応える
-   今後の機能拡張により、さらなる可能性が広がる

# 参考情報

-   [Azure AI エージェント サービスとは](https://learn.microsoft.com/ja-jp/azure/ai-services/agents/overview)
-   [Unlocking AI-Powered Automation with Azure AI Agent Service](https://techcommunity.microsoft.com/blog/azure-ai-services-blog/unlocking-ai-powered-automation-with-azure-ai-agent-service/4372041)
-   [OpenAI Python API library](https://github.com/openai/openai-python)
-   [Azure AI Projects client library for Python](https://github.com/Azure/azure-sdk-for-python/tree/main/sdk/ai/azure-ai-projects)
-   [Azure Functions - Using Azure Functions to enable function calling from Azure AI Agent service](https://github.com/Azure-Samples/azure-functions-ai-services-agent-python)
