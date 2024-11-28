---
title: ""
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## （Tips）東日本リージョンでは Office 系のファイルをサポートしていないって本当？

Studio で東日本リージョンにデプロイされた Document Intelligence を利用する際に、プレビューの API バージョンを選択できないようになっています（2024 年 11 月 28 日現在の画面）。

![alt text](/images/japaneast-di-support-preview-api/check-supported-api-versions.png)

[ドキュメント](https://learn.microsoft.com/ja-jp/azure/ai-services/document-intelligence/prebuilt/layout?view=doc-intel-4.0.0&tabs=sample-code#input-requirements-v4)から引用した下表によると、Read (読み込み) モデルは GA の API バージョンでサポートされているようですが、Layout モデルはプレビューの API バージョンでサポートされているようです。ということは、東日本リージョンでは Layout モデルを使って Office 系のファイルを処理することができないということでしょうか？

| モデル               | PDF | 画像: JPEG/JPG、PNG、BMP、TIFF、HEIF | Microsoft Office: Word (DOCX)、Excel (XLSX)、PowerPoint (PPTX)、HTML |
| -------------------- | --- | ------------------------------------ | -------------------------------------------------------------------- |
| 読み込み             | ✔   | ✔                                    | ✔                                                                    |
| Layout               | ✔   | ✔                                    | ✔ (2024-07-31-preview、2024-02-29-preview、2023-10-31-preview)       |
| 一般的なドキュメント | ✔   | ✔                                    |                                                                      |
| 事前構築済み         | ✔   | ✔                                    |                                                                      |
| カスタム抽出         | ✔   | ✔                                    |                                                                      |
| カスタム分類         | ✔   | ✔                                    | ✔ (2024-07-31-preview、2024-02-29-preview)                           |

### 否！東日本リージョンでもプレビューの API バージョンを使って Office 系のファイルを処理できる
