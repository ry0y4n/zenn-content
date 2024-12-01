---
title: "【Document Intelligence】東日本リージョンでは Office 系のファイルをサポートしていないって本当？"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [azure, documentintelligen, mslearn]
published: true
---

# 結論

東日本リージョンにデプロイされた Document Intelligence はプレビューの API バージョンを使って Office 系のファイルを処理することができます。

# 前提

以下のドキュメントによると、「読み込み」、「Layout（レイアウト）」、「カスタム分類」の 3 つのモデルが Office 系のファイルをサポートしています。

これらのうち「レイアウト」と「カスタム分類」は、プレビューの API バージョンで Office 系のファイルをサポートしています（2024 年 12 月 01 日現在）。
https://learn.microsoft.com/ja-jp/azure/ai-services/document-intelligence/prebuilt/read?view=doc-intel-3.1.0&tabs=sample-code#input-requirements

| モデル               | PDF | 画像: (JPEG/JPG、PNG、BMP、TIFF、HEIF) | Microsoft Office: (Word (DOCX)、Excel (XLSX)、PowerPoint (PPTX)、HTML) |
| -------------------- | --- | -------------------------------------- | ---------------------------------------------------------------------- |
| 読み込み             | ✔   | ✔                                      | ✔                                                                      |
| Layout               | ✔   | ✔                                      | ✔ (2024-07-31-preview、2024-02-29-preview、2023-10-31-preview)         |
| 一般的なドキュメント | ✔   | ✔                                      |                                                                        |
| 事前構築済み         | ✔   | ✔                                      |                                                                        |
| カスタム抽出         | ✔   | ✔                                      |                                                                        |
| カスタム分類         | ✔   | ✔                                      | ✔ (2024-07-31-preview、2024-02-29-preview)                             |

## 東日本リージョンで Office 系のファイルを処理することが出来ないと思ってしまう理由

何が問題なのかというと、Document Intelligence Studio で機能をテストしようとする際の挙動です。

Document Intelligence Studio で東日本リージョンにデプロイされた Document Intelligence を利用する際に、プレビューの API バージョンを選択できないようになっています（2024 年 12 月 01 日現在の画面）。

![alt text](/images/japaneast-di-support-preview-api/check-supported-api-versions.png)

もちろん、その状態で Office 系のファイルをアップロードしようとすると、以下のようなエラーが発生します。

![alt text](/images/japaneast-di-support-preview-api/cant-upload-office-file.png)

これは、東日本リージョンでプレビュー API バージョンをサポートされていないために発生しているエラーです。

以下のドキュメントでは最新の API バージョン（2024-07-31-preview）をサポートしているリージョンの一覧が記載されています。
https://learn.microsoft.com/ja-jp/azure/ai-services/document-intelligence/versioning/sdk-overview-v4-0?view=doc-intel-4.0.0&tabs=csharp

![alt text](/images/japaneast-di-support-preview-api/preview-api-version-support-list.png)

## では、東日本リージョンで Office 系のファイルを処理することが出来ないのか？

否！東日本リージョンでもプレビューの API バージョンを使って Office 系のファイルを処理できる。

というかコードを書いてみたら動きました。

今回は、Python の SDK を使って Office 系のファイルを処理するコードを書いてみました。

```Python
from azure.core.credentials import AzureKeyCredential
from azure.ai.documentintelligence import DocumentIntelligenceClient
from azure.ai.documentintelligence.models import AnalyzeResult
import os
from dotenv import load_dotenv
load_dotenv('.env')

endpoint = os.environ["FORM_RECOGNIZER_ENDPOINT"]
key = os.environ["FORM_RECOGNIZER_KEY"]
path_to_sample_documents ="sample-layout.xlsx"
# path_to_sample_documents = "sample-layout.docx"

# 東日本リージョンの Document Intelligence を使ってクライアントを作成
document_analysis_client = DocumentIntelligenceClient(
    endpoint=endpoint, credential=AzureKeyCredential(key)
)

# Office 系のファイルを読み込んで Document Intelligence で処理
with open(path_to_sample_documents, "rb") as f:
    poller = document_analysis_client.begin_analyze_document(
        "prebuilt-layout", analyze_request=f, content_type="application/octet-stream"
    )

result: AnalyzeResult = poller.result()
print("modelId: {}".format(result.model_id))
print("apiVersion: {}".format(result.api_version))
print("content: {}".format(result.content))
```

出力は以下のようになります。

```Shell
$ uv run layout-test.py

modelId: prebuilt-layout
apiVersion: 2024-07-31-preview
content: Sheet1
page1
Sheet2
page2
```

解析結果の中で使用した API バージョンを確認することができまして、2024-07-31-preview が使われていることがわかります。

## SDK で使用している API バージョン

以下のドキュメントにて、各言語の SDK で使用している API バージョンを確認することができます。
https://learn.microsoft.com/ja-jp/azure/ai-services/document-intelligence/versioning/sdk-overview-v4-0?view=doc-intel-4.0.0

例えば、Python であれば `1.0.0b2` 以降のバージョンの SDK を使うことで、2024-07-31-preview の API バージョンを使うことができます。

![alt text](/images/japaneast-di-support-preview-api/python-sdk-versions.png)

私の環境では `1.0.0b4` の SDK を使っていますので、2024-07-31-preview の API バージョンを使うことができたということですね。

# 残る 2 つの疑問について

以下について、予想を述べます（完全なる個人的妄想です）。

## なぜ Studio でプレビューの API バージョンを選択できないのか？

以下のようなことが起きているのではと予想しています。

-   Document Intelligence (API): 東日本リージョンにデプロイされた Document Intelligence はプレビューの API バージョンをサポートしている
-   Document Intelligence Studio: 東日本リージョンは、プレビューの API バージョンをサポートしていないと認識している

前提として、Document Intelligence Studio は Document Intelligence の API を叩く Web アプリケーションです。実際の API の仕様と別に Studio 側で「リージョン」とそのリージョンがサポートしている「API バージョン」の情報を持っていて、その情報を元に API バージョンの選択可否を実装していると思われます。今回は、その情報が古くなっているせいで、プレビューの API バージョンを選択できないようになっているのではないかと考えています。

## 公式ドキュメントにはプレビューの API バージョンをサポートしているリージョン一覧に東日本リージョンが含まれていないが、実際には使えるのはなぜか？

これもおそらくドキュメントの情報が古いためだと考えています。現実として、Document Intelligence の API はプレビューの API バージョンをサポートしているため、東日本リージョンでも Office 系のファイルを処理することができると思われます。

これについて、私の見解が正しければ修正するように PR を出したので、回答を待ちたいと思います（進捗あり次第更新します）。
https://github.com/MicrosoftDocs/azure-ai-docs/pull/118

# まとめ

-   東日本リージョンで Office 系のファイルを処理することは可能
-   しかし、Studio では使えないので、コードを書いて使うことをおすすめします
-   ドキュメントも古い情報が含まれている可能性があるので、やっぱり実際に試してみることをおすすめします

# 参考

-   [SDK ターゲット: REST API 2024-07-31-preview](https://learn.microsoft.com/ja-jp/azure/ai-services/document-intelligence/versioning/sdk-overview-v4-0?view=doc-intel-4.0.0)
-   [Document Intelligence 読み取りモデル](https://learn.microsoft.com/ja-jp/azure/ai-services/document-intelligence/prebuilt/read?view=doc-intel-3.1.0&tabs=sample-code#input-requirements)
