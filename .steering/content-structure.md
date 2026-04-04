# Content Structure

## リポジトリ構成

```
zenn-content/
├── articles/          # 記事（Markdown + Front Matter）
│   └── *.md           # ハッシュベースのファイル名（例: 5330b19412687ee0b435.md）
├── books/             # ブック（長編コンテンツ）
├── .claude/
│   └── rules/         # Claude Code向け行動規約
├── .steering/         # プロジェクト背景知識
└── CLAUDE.md          # エントリポイント
```

## Zennプラットフォームとの連携

- publishedがtrueの記事は、GitHubへのpush時にZennへ自動同期される
- ビルドプロセスやテストフレームワークは存在しない（純粋なMarkdownベース）
- 記事のプレビューは `npx zenn preview` でローカル確認

## 記事ファイルの構造

各記事は以下の構成:
1. **Front Matter**（YAML）: メタデータ（タイトル、絵文字、タイプ、トピック、公開状態）
2. **本文**（Markdown）: 記事コンテンツ（日本語）

## 開発フロー

1. `npx zenn preview` でローカルサーバー起動
2. `npx zenn new:article` で新規記事のテンプレート生成
3. `articles/` 配下のMarkdownを編集
4. ブラウザでプレビュー確認
5. featureブランチからPRを作成（Git運用ルールは `.claude/rules/git-workflow.md` 参照）
