---
title: "【Slack + Bitbucket連携】issueコメントやプルリク通知をチャンネルに流す方法"
emoji: "🔔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [slack, bitbucket]
published: true
---

# SlackにBitbucket Cloudアプリを追加
まずはSlackに`Bitbucket Cloud`アプリを追加します。
https://www.atlassian.com/partnerships/slack?tab=bitbucket

![](https://storage.googleapis.com/zenn-user-upload/l0efcqnl4j7sbjtvxbqkby3qhl4d)

## Slack対象チャンネルにBitbucket Cloudアプリを招待
Slackの対象チャンネルに移動し、以下コマンドでBitbucket Cloudアプリを招待します。
```
/invite @Bitbucket
```
以下画像のような表示が出ればOKです。
![](https://storage.googleapis.com/zenn-user-upload/g80ntgeqyji24n9lwtw6rpwcr1ax)


# 購読対象リポジトリの接続
以下コマンドで購読対象のBitbucketリポジトリと接続します。
```
/bitbucket connect <your-repository-url>
```

`Create subscription` ボタンをクリック。
![](https://storage.googleapis.com/zenn-user-upload/q004ah3bcy4gvcob6bxz2ep166vp)

ブラウザでBitbucketリポジトリのページが表示されるので、ワークスペースとチャンネルを確認して`Add`ボタンをクリック。
![](https://storage.googleapis.com/zenn-user-upload/dfcloh5u00wy6kbac5qv7zev9qsh)

Slackチャンネルに購読の開始通知が流れます。
![](https://storage.googleapis.com/zenn-user-upload/s6gxiyhnwc7sj36fw9uis1iyrib8)

試しにissueに変更を加えてみると、無事に変更通知が流れてきました。
![](https://storage.googleapis.com/zenn-user-upload/ru0farv1x9f3egmd8bwcer4lyph8)

やったぜ。
# 通知内容の詳細設定
Bitbucket上の設定画面で通知内容の詳細設定を変更できます。
![](https://storage.googleapis.com/zenn-user-upload/oy6e0tdzrkmbsy23cddbmxmc0nlk)
不要な通知は除外するなど、運用に応じたカスタマイズが可能です。

# 参考
[Integrate Bitbucket Cloud with Slack | Bitbucket Cloud | Atlassian Support](https://support.atlassian.com/bitbucket-cloud/docs/integrate-bitbucket-cloud-with-slack/ )