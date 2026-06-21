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

## 表記・用語の方針

- **Vanilla JS** … 素の JavaScript を指す表記は「Vanilla JS」で統一（「バニラJS」「バニラ JavaScript」は使わない）。初出付近で語源解説記事を添える: https://zenn.dev/seya/articles/66809b3d59151c
- **表記ゆれの統一** … 同じ対象を指す語は記事内で1つに統一する。やむを得ず複数表現を使う場合は、初出で「正式名（別名）」の形で定義してから以降を統一（例: 「資産構成ドーナツ（円グラフ）」と定義し、以降は「ドーナツ」）。
- **専門用語の補足** … 想定読者に馴染みの薄い用語（トンマナ、ヒーロー、mixin、dc-runtime 等）は初出で簡単な補足を入れる。短い補足はインライン括弧書き、まとまった解説や参照リンクは Zenn のメッセージ記法（`:::message`）やトグル（`:::details`）を活用する。
  - Zenn記法リファレンス: https://zenn.dev/zenn/articles/markdown-guide#%E3%83%A1%E3%83%83%E3%82%BB%E3%83%BC%E3%82%B8
  - dc-runtime（`<x-dc>` カスタム要素ベースの軽量ランタイム。`support.js` はその生成物で `// GENERATED from dc-runtime/src/*.ts` と明記。公開ドキュメントは未確認のため出自は断定せず、リポジトリ内の事実と一次体験に基づいて記述する）の補足で添える「ランタイムとは」解説: https://zenn.dev/tenchijin/articles/20240618_runtime
- **Claude Design の初出解説** … Claude Design に言及する記事では初出で機能の一言説明と公式ヘルプを添える:
  - https://support.claude.com/ja/articles/14604416-claude-design%E3%82%92%E5%A7%8B%E3%82%81%E3%82%8B
  - https://support.claude.com/ja/articles/14604397-claude-design%E3%81%A7%E3%83%87%E3%82%B6%E3%82%A4%E3%83%B3%E3%82%B7%E3%82%B9%E3%83%86%E3%83%A0%E3%82%92%E3%82%BB%E3%83%83%E3%83%88%E3%82%A2%E3%83%83%E3%83%97%E3%81%99%E3%82%8B

## 過去記事の活用

新規記事は過去の自著記事の知見を引き継いでいることが多い。関連する土台（CLAUDE.md＋`.claude/rules/` 運用、GitHub Pages × GitHub Actions の静的サイト構成など）に触れる箇所では、該当する過去記事をカードリンクで関連付ける。

- Claude Code × CLAUDE.md＋rules でコミュニティポータルサイトを構築した記録: https://zenn.dev/easy_easy/articles/89318e4d0acb4f
