---
description: Git操作（commit, push, branch, PRなど）を行うすべての場面で適用
---

# Git Workflow Rules

## 絶対原則

mainブランチへの直接コミットは原則禁止。すべての変更はfeatureブランチ経由でPRを通す。

**例外**: ユーザーが会話中に「mainに直接コミットしてよい」と明示的に指示した場合は、その指示に従う。

## コミット前の必須手順

**すべてのgit commitの前に、以下を必ず実行すること：**

1. `git branch --show-current` で現在のブランチを確認
2. mainブランチにいる場合は **即座に中断** し、featureブランチに切り替える
3. 既存ブランチの確認: `git branch -a`
4. 関連するfeatureブランチがあればそれを使用、なければ新規作成

## ブランチ命名規則

- 形式: `feature/簡潔な説明`（例: `feature/claude-code-article`）
- 新規作成: `git checkout -b feature/description`

## コミットからPRまでの流れ

```bash
# 1. ブランチ確認（必須）
git branch --show-current

# 2. featureブランチへ切り替え（mainにいる場合）
git checkout -b feature/description

# 3. 変更をコミット
git add <対象ファイル>
git commit -m "メッセージ"

# 4. プッシュ
git push -u origin feature/description

# 5. PR作成
gh pr create -a unsolublesugar -l enhancement
```

## PR作成ルール

- `gh pr create -a unsolublesugar -l enhancement` を使用（オーナーアサイン + enhancementラベル自動付与）
- ベースブランチは `main`（リモートのデフォルトブランチ）。`gh pr create` 時に `--base master` を付けない（`master` も存在するため誤指定に注意）
- PRの説明とテストプランを含めること

### PR・Issue本文の自動リンク対策

GitHub はPR/Issue **本文** 内の `#数字` や `@ユーザー名` を、リポジトリ内のIssue/PRやGitHubアカウントへ自動リンク（＋メンション通知）してしまう。イベント名の「#78」やSNSアカウントの「@handle」が無関係な対象へ繋がるため、本文では次のいずれかで無効化する。

- **`#数字`**（例: イベント回次の `#78`）
  - 正確なURLがある場合はリンク化する（例: `[#78](https://easy2.connpass.com/event/395246/)`）
  - 汎用的な参照はバックスラッシュでエスケープする（例: `\#77`・`\#76`）
- **`@ユーザー名`**（SNSハンドル等、GitHubアカウントでないもの）
  - バッククォートのコード表記にする（例: `` `@paseri_kurosawa` ``）、または正しいプロフィールURLへリンク化する
- **PR/Issueタイトル** はGitHubがプレーンテキスト扱いで自動リンクしないため、対応不要
- Markdown見出し（記事本文の `# LT1:` 等）はGitHub上の自動リンク対象外なので変更しない
