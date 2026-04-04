---
description: 記事・ブックの作成・編集時に適用（articles/, books/配下のMarkdownファイル）
paths: ["articles/**/*.md", "books/**/*.md"]
---

# Content Guidelines

## Front Matter仕様

```yaml
---
title: "記事タイトル"
emoji: "📝"
type: "tech"  # tech: 技術記事 / idea: アイデア記事
topics: [topic1, topic2]  # 最大5つまで（超過するとエラー）
published: true  # false: 下書き
publication_name: "publication_name"  # 任意
---
```

## 厳守事項

- **topicsは最大5つ**。超過するとZenn側でエラーになる
- **URLは独立した行に配置**。ZennはスタンドアロンURLをカードビューとして自動展開する
- Front Matterが不正な記事は無効扱いになる

## コンテンツタイプ

| type | 用途 |
|------|------|
| `tech` | 技術的な内容の記事 |
| `idea` | 意見・アイデア系の記事 |
