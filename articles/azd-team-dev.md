---
title: "Azure Developer CLI を使った Azure システムの開発"
emoji: "🐷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [azure]
published: false
publication_name: microsoft
---

# Azure Developer CLI とは

# 開発事始め

## プロジェクトの作成

## リソースのプロビジョン

## アプリのデプロイ

# 複数の環境での開発

```bash
azd env list
```

```bash
azd env select <env-name>
```

# チーム開発時の運用

`azd provision` でリソースをプロビジョンすると、ルート フォルダーに `.azure` フォルダが作られる。その中に環境変数（`.env`）が格納され、それを参照してローカルでの開発を行う。

では、プロビジョニング担当以外の開発者はどうするか？その場合は、以下のコマンドを行って `.azure` フォルダを同期することが可能である。

```bash
azd env refresh
```
