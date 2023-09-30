---
title: "【新公式GitHubアクション】「Create GitHub App Token」を試してみよう"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["github", "githubactions", "yaml", "cicd"]
published: true
---

# 知らない間に出てました

ワークフロー内での権限管理に[GitHub Apps](https://docs.github.com/ja/apps/overview)を使う時に，そのアクセストークンを発行するための**公式**アクションが出てました．

> GitHub ActionsでGitHub Appsを使うと何が嬉しいのか？どういう時に使うのか？については，[「GitHub Appsトークン解体新書：GitHub ActionsからPATを駆逐する技術」](https://zenn.dev/tmknom/articles/github-apps-token)がよくまとまっています．


https://github.com/actions/create-github-app-token

v1.0.0のリリースが2023年6月9日，現在がv1.2.1（2023年8月30日）ということで中々新しいアクションですね．

# 今まではどうしていたのか？

https://github.com/tibdex/github-app-token

↑これを使っていました．

以下が使用例です．GitHub CLIでシークレットを設定するのに"Secrets"のwrite権限をつけたGitHub Appsをインストールし，リポジトリのシークレットに設定された**APP_ID**と**PRIVATE_KEY**を使ってトークンを生成し**GITHUB_TOKEN**にパスしてます．これがないと`gh secret set ~~`が権限エラーで失敗します．GitHub Apps便利〜．

```yaml
jobs:
  job:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: tibdex/github-app-token@v2
        id: generate-token
        with:
          app_id: ${{ secrets.APP_ID }}
          private_key: ${{ secrets.PRIVATE_KEY }}
      - name: Set secrets
        env:
          GITHUB_TOKEN: ${{ steps.generate-token.outputs.token }}
        run: |
          gh secret set MYSECRET --body "SECRET VALUE"
```

## tibdex/github-app-tokenじゃダメなのか？

### サードパーティーの危うさ

サードパーティーアクションである以上，今後メンテナンスがされなくなりアクションの正常性が保守されなくなる可能性があります（公式だから絶対安心というわけではないですが安心度が違います）．

例えば，[tibdex/github-app-token](https://github.com/tibdex/github-app-token)以前によく使われていた[getsentry/action-github-app-token](https://github.com/getsentry/action-github-app-token)は，node16へのアップデートコミットを除けば，最終の実装コミットは2021年8月です．「GitHub Appsのトークンを払い出す」というミニマムなアクションなので更新要素が少ないのは理解できますが，更新頻度が少ないのは少し「おっ」と思いますよね．IssueやPRもしっかり溜まってますし．

### 企業のセキュリティルール

企業によってはサードパーティーのアクションが使えない場合があるかもしれません．そこまでじゃなくてもいくつかの制限がある企業を知っています．

制限の例で言うと，ソースコードを読んでセキュリティインシデントが起きないと確信できたものだけを，そのバージョンのコミットハッシュで固定して使用することを許可されている等です．

```yaml
# コミットハッシュで固定
uses: tibdex/github-app-token@3eb77c7243b85c65e84acfa93fdbac02fb6bd532
```


上記の理由で，公式アクションを使った方が安全・安心かつ使い勝手も良いのではないでしょうか．

# 実際に使ってみよう！

以下がactions/create-github-app-tokenを使って，前述の使用例を書き換えたものです．

```yaml
jobs:
  set-secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/create-github-app-token@v1
        id: generate-token
        with:
          app_id: ${{ secrets.APP_ID }}
          private_key: ${{ secrets.PRIVATE_KEY }}
      - name: Set secrets
        env:
          GITHUB_TOKEN: ${{ steps.generate-token.outputs.token }}
        run: |
          gh secret set MYSECRET --body "SECRET VALUE"
```

お？何が変わったんだ？

```diff yaml
jobs:
  set-secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
-     - uses: tibdex/github-app-token@v2
+     - uses: actions/create-github-app-token@v1
        id: generate-token
        with:
          app_id: ${{ secrets.APP_ID }}
          private_key: ${{ secrets.PRIVATE_KEY }}
      - name: Set secrets
        env:
          GITHUB_TOKEN: ${{ steps.generate-token.outputs.token }}
        run: |
          gh secret set MYSECRET --body "SECRET VALUE"
```

これだけ！これだけで公式アクションへの挿げ替えを完了することができます．これならマイグレーションのコストもほぼ無いに等しいですね．私も今後はこっちを使うようにしたいと思います．

---

また，GitHub Appsの作成やインストール手順などは，DeveloperIOさんの記事「[GitHub Apps + GitHub Actionsで必要なアクセス権限のみ付与した一時的なアクセストークンを発行する](https://dev.classmethod.jp/articles/getting-an-access-token-with-only-the-necessary-permissions-on-github-appsgithub-actions/)」が非常に親切で分かりやすいのでぜひご参考にしてください．


# まとめ

- 公式アクション「[Create GitHub App Token](https://github.com/actions/create-github-app-token)」を使っていこう
- 挿げ替えは超簡単
- GitHub Appsはやっぱり便利
