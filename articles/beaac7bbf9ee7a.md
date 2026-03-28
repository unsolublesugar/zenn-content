---
title: "「あとで読む（読まない）」を仕組みで解決する — AI駆動の個人開発記"
emoji: "💧"
type: "tech"
topics: [python, claudecode, ai, githubactions, 個人開発]
published: true
---

## はじめに

Raindrop.ioに「あとで読みたい記事」を溜めているが、読まないまま埋もれていく一方だった。必要なのは気合いではなく仕組みだと思い、ChatGPTで仕様を固め、Claude Codeで実装した。

https://github.com/unsolublesugar/tsuyu-mi

Raindropの特定コレクションを定期的に走査し、記事本文をAIで要約、優先度つきのダッシュボードとして表示するツールだ。

![Tsuyu-miダッシュボード](https://storage.googleapis.com/zenn-user-upload/407aa689aab1-20260328.webp)
*[Tsuyu-mi ダッシュボード](https://unsolublesugar.github.io/tsuyu-mi/)*

この記事では、仕様策定から実装、v1.0.0リリースまでの開発プロセスを紹介する。

## 仕様策定 — ChatGPTで仕様書を用意

開発の出発点は、ChatGPTとの仕様相談だった。やりたいことの大枠を伝え、仕様書の骨格を整理してもらった。

![](https://storage.googleapis.com/zenn-user-upload/f6730e930d38-20260328.png)

目的は明確で、Raindrop.ioに溜めた「あとで読む」記事を、全文を読む前に一次判定できる状態にすること。

- **今読むべきか**（high）
- **後回しでよいか**（medium）
- **読まずに捨ててよいか**（low）

この3段階の優先度をAIに判定させ、要約つきで並べるダッシュボードを作る方針を固めた。

ChatGPTで整理した仕様をもとに、いわゆる仕様駆動開発のアプローチで実装前に仕様を明文化し、それをベースに開発を進める形をとった。

以下がChatGPTに作ってもらった実装計画の全容だ。

:::details ChatGPTが作成した実装計画（Raindrop Collection Summarizer）
# Raindrop Collection Summarizer

Raindrop.io の特定コレクションを定期取得し、保存記事の本文を抽出して AI で短く要約し、HTML 一覧として出力するためのバッチツールです。

## 目的

Raindrop に溜めた「あとで読む」記事を、全文を読む前に一次判定できる状態にすることを目的とします。

このツールは、記事を完全に読む代替ではなく、以下の判断をしやすくするための前処理を自動化します。

- 今読むべきか
- 後回しでよいか
- 読まずに捨ててよいか

## スコープ

初期リリースでは、以下の範囲を対象にします。

- Raindrop.io の **単一コレクション**
- 定期取得または手動実行
- URL 先本文の抽出
- AI による日本語要約
- HTML 一覧出力
- JSON による永続化と差分管理

初期リリースでは以下は対象外、または限定対応とします。

- 動画コンテンツの要約
- 複数コレクション横断処理
- DB 導入
- Web UI からの編集
- note への自動投稿
- Playwright などの重いブラウザ自動化

---

## 要件サマリ

### 固定条件

- 対象コレクションは 1 つ
- 本文抽出失敗時は簡易要約を試す
- 簡易要約も難しければスキップ
- 優先度は 3 段階
- 1 回の要約件数上限は 30 件
- 出力の主用途は HTML
- Markdown 出力は将来追加しやすい構造にする
- 要約出力は日本語
- 英語記事も可能な限り日本語で要約
- 動画は初期リリースの対象外。ただし判別はしたい

---

## ユースケース

### 想定ユースケース

- 数日おきに Raindrop の「あとで読む」コレクションを取得する
- 未処理記事だけを対象に本文抽出する
- AI で短く日本語要約する
- HTML 一覧を生成し、読む/保留/捨てる判断材料にする
- 将来的に GitHub Actions + GitHub Pages で自動運用する

### 非ゴール

- 記事の完全要約や網羅的読解
- ファクトチェックを伴う厳密な記事理解
- 動画や音声の内容理解
- クローズドな paywall 記事への完全対応

---

## 全体アーキテクチャ

```
Raindrop API
  ↓
fetch raindrops
  ↓
diff check / state management
  ↓
fetch URL content
  ↓
extract article body
  ↓
fallback simple summary input generation
  ↓
LLM summarization
  ↓
store article JSON
  ↓
build HTML output
```

### 処理ステップ

1. Raindrop.io API から対象コレクションの記事一覧を取得
2. 既処理記事と比較し、新規または再処理対象だけ抽出
3. 各記事 URL の本文抽出を試みる
4. 本文抽出失敗時はタイトル・excerpt・URL などから簡易要約用入力を構成
5. AI で日本語要約を生成
6. 記事単位の JSON を保存
7. JSON をもとに HTML 一覧を生成

---

## 推奨実装方針

### 基本方針

* Python ベース
* JSON ファイルストレージ
* HTML を主出力
* ローカル実行を先に成立させる
* その後 GitHub Actions に載せ替え可能な構造にする

### 理由

* 差分管理や再実行がしやすい
* daily-tech-news 的な定期収集の発想を流用しやすい
* HTML 生成まで静的ファイルで完結できる
* 初期段階でデバッグしやすい

---

## ディレクトリ構成案

```
raindrop-collection-summarizer/
├─ README.md
├─ pyproject.toml
├─ .env.example
├─ src/
│  ├─ main.py
│  ├─ config.py
│  ├─ models.py
│  ├─ logging_util.py
│  ├─ raindrop_client.py
│  ├─ state_store.py
│  ├─ content_fetcher.py
│  ├─ content_extractor.py
│  ├─ summarizer.py
│  ├─ article_repository.py
│  ├─ html_builder.py
│  └─ utils/
│     ├─ hashing.py
│     ├─ text.py
│     └─ time.py
├─ prompts/
│  ├─ summarize_full.txt
│  └─ summarize_fallback.txt
├─ data/
│  ├─ raw/
│  │  └─ latest_raindrops.json
│  ├─ articles/
│  │  └─ {raindrop_id}.json
│  └─ output/
│     ├─ articles.json
│     └─ latest_summary.json
├─ state/
│  └─ index.json
├─ docs/
│  ├─ index.html
│  ├─ styles.css
│  └─ app.js
└─ .github/
   └─ workflows/
      └─ build.yml
```

---

## 環境変数

`.env` で管理する想定です。

```env
RAINDROP_TOKEN=
RAINDROP_COLLECTION_ID=
LLM_PROVIDER=openai
LLM_API_KEY=
LLM_MODEL=
MAX_SUMMARIZE_PER_RUN=30
REQUEST_TIMEOUT_SECONDS=20
USER_AGENT=RaindropSummarizer/0.1
OUTPUT_DIR=docs
DATA_DIR=data
STATE_DIR=state
LOG_LEVEL=INFO
```

---

## データモデル

### 記事メタデータ

```json
{
  "raindrop_id": 123456789,
  "collection_id": 111111,
  "title": "Example title",
  "url": "https://example.com/article",
  "domain": "example.com",
  "created_at": "2026-03-25T00:00:00Z",
  "tags": [],
  "excerpt": "short excerpt"
}
```

### 処理済み記事 JSON

```json
{
  "raindrop_id": 123456789,
  "title": "記事タイトル",
  "url": "https://example.com/article",
  "topic": "AI と検索体験の変化",
  "summary_3lines": ["1行目", "2行目", "3行目"],
  "priority": "medium",
  "read_now_reason": "...",
  "defer_reason": "...",
  "drop_candidate": false,
  "keywords": ["AI", "検索", "プロダクト"],
  "model_provider": "openai",
  "model_name": "gpt-4.1-mini",
  "summarized_at": "2026-03-25T08:31:00Z"
}
```

### state/index.json

```json
{
  "last_run_at": "2026-03-25T08:31:00Z",
  "items": {
    "123456789": { "status": "summarized", "content_hash": "sha256:..." },
    "987654321": { "status": "skipped", "reason": "summary_input_unavailable" }
  }
}
```

---

## 状態管理

記事ごとに以下の状態を持ちます。

`pending` → `fetched` → `extracted` → `summarized`
（失敗時: `skipped` / `failed`）

---

## 実装ステップ

### Phase 1: MVP の最小成立

Raindrop から取得し、本文抽出し、AI 要約して HTML を出せる状態。

### Phase 2: 運用強化

failed / skipped の理由精緻化、再試行コマンド、フィルタ UI、ログ整備、HTML デザイン改善。

### Phase 3: 自動運用

GitHub Actions ワークフロー、secrets 設定、docs 自動更新、GitHub Pages 公開。

---

## Claude Code への実装依頼メモ

1. まずは Python で動く MVP を優先
2. ローカル実行で完結すること
3. HTML 出力を主目的にすること
4. JSON を正本にすること
5. 本文抽出失敗時は簡易要約にフォールバックすること
6. 英語記事でも日本語要約すること
7. 動画 URL は初期リリースではスキップすること
8. 後から GitHub Actions に載せやすい構成にすること
9. LLM provider は差し替え可能にすること
10. 過剰に作り込みすぎず、まずは end-to-end で動くことを優先すること
:::

この実装計画をClaude Codeに渡して開発を進めた。Claude Codeはこの仕様をもとに、各機能領域ごとの仕様ドキュメントを `docs/specs/` に整理してくれた（詳細はStep 0で後述）。

## すべてClaude Codeでやった

今回の開発で特徴的だったのは、コードの実装だけでなく、開発に関わるほぼすべての作業をClaude Codeで完結したことだ。

- **仕様ドキュメント整備**: 実装計画から8つの仕様書を `docs/specs/` に配置
- **Skill・Rules作成**: `/update-specs` スキルやワークフロー規約の整備
- **技術選定の相談**: LLMプロバイダーの料金比較やAPIキー取得手順の案内
- **Issue作成**: 各Stepの実装タスクをIssueとして起票
- **PR作成**: ブランチ作成からPR説明文の記述まで
- **サービス命名**: リポジトリ名のリネームと一括反映
- **README**: 英語版 + 日本語版の作成、バッジ追加
- **セキュリティ監査**: プロジェクト全体の精査と改善提案
- **ブランチ保護**: GitHub Rulesetの設定
- **Git履歴の書き換え**: 機密データの完全削除
- **リリースノート**: v1.0.0の全機能をカテゴリ分けして整理

## Claude Code × 仕様駆動開発ワークフロー

今回の開発では、Claude Codeを単なるコード生成ツールとしてではなく、開発ワークフロー全体のパートナーとして活用した。具体的には、`CLAUDE.md・`Skill・Rules・`.steering/` の4層でプロジェクトのコンテキストを共有するのが、今回の開発で得た知見のひとつだ。

### CLAUDE.mdによるコンテキスト共有

Claude Codeでは、プロジェクトルートに `CLAUDE.md` を配置することで、プロジェクトのコンテキストをAIに共有できる。会話の冒頭で毎回説明しなくても、プロジェクトの概要や使用技術、コマンド体系をClaude Codeが把握した状態で作業を開始できる。

https://code.claude.com/docs/ja/memory

本プロジェクトでは以下のように記述している。

:::details CLAUDE.md
```markdown
# CLAUDE.md

必ず日本語で回答してください。

## Project

Tsuyu-mi — Raindrop.io コレクションの記事を取得・本文抽出・AI 要約し、HTML 一覧を出力するバッチツール。

## Tech Stack

Python 3.11+ / httpx / pydantic / trafilatura / readability-lxml / jinja2 / click / rich

## Commands

- Install: `pip install -e ".[dev]"`
- Run: `python -m src.main run`
- Test: `pytest`
- Lint: `ruff check src/ tests/`

## References

- 詳細ルール: `.claude/rules/` (自動読み込み)
- アーキテクチャ・設計方針: `.steering/`
- 仕様書: `docs/specs/`
```
:::

非常にコンパクトだが、これだけで言語指定、プロジェクト概要、技術スタック、コマンド体系、関連ドキュメントの所在がClaude Codeに伝わる。CLAUDE.mdはあくまでエントリポイントとして簡潔にまとめ、詳細はSkill・Rules・`.steering/` に委ねる構成にしている。

### Skillの活用

Skillは `.claude/skills/` に配置するカスタムスラッシュコマンドで、繰り返し行う操作をワンコマンドで実行できる。`SKILL.md` にプロンプトを記述しておくと、`/skill-name` で呼び出せる仕組みだ。なお、以前の「カスタムコマンド（`.claude/commands/`）」機能は現在Skillに統合されている。

https://code.claude.com/docs/ja/skills

仕様駆動開発を支えるために、カスタムSkill `/update-specs` を作成した。

:::details SKILL.md — update-specs
```
---
name: update-specs
description: コード変更に関連する docs/specs/ の仕様書を更新する。コード変更の完了時に自動で実行する。
context: fork
---

変更されたファイルに関連する `docs/specs/` 配下の仕様書を特定し、更新案を提示する。

## 手順

1. `git diff --name-only HEAD` で変更されたファイル一覧を取得する（Python ファイル、およびファイルのリネーム・移動・削除を含む）
2. `docs/specs/` 配下の各仕様書を読み込み、変更ファイルのパスや関連モジュール名が含まれるものを特定する。対応表:
   - `src/config.py`, `src/logging_util.py` → `01_overview.md`
   - `src/models.py` → `02_data_models.md`
   - `src/raindrop_client.py` → `03_api_integration.md`
   - `src/content_fetcher.py`, `src/content_extractor.py` → `04_content_extraction.md`
   - `src/summarizer.py`, `prompts/` → `05_summarization.md`
   - `src/html_builder.py`, `docs/` (specs以外) → `06_html_output.md`
   - `src/state_store.py`, `src/article_repository.py` → `07_state_management.md`
   - `src/main.py` → `08_cli_interface.md`
3. 関連する仕様書が見つからない場合、新機能の可能性がある。変更ファイルのパスとディレクトリ構造から機能名を推定し、`docs/specs/` への新規仕様書の作成を提案する
4. 関連する仕様書と変更された Python ファイルの内容を読み込む
5. 仕様書の各セクション（データモデル、フロー、ビジネスルール等）が変更内容と整合しているか確認する
6. 乖離がある場合は、セクションごとに具体的な更新案を提示する
7. ファイルのリネーム・移動・削除があった場合、`git diff --diff-filter=RD --name-status HEAD` で検出し、全仕様書・README.md・CLAUDE.md・`.steering/` 内に旧パスの記載が残っていないか確認する。残っていれば新パスへの更新（または削除済みの旨の反映）を提示する
8. `README.md` を読み込み、変更内容に関連する記載（機能一覧、技術スタック、セットアップ手順等）が最新と合っているか確認する。更新が必要であれば合わせて提示する
9. `CLAUDE.md` を読み込み、変更内容に関連する記載（Tech Stack、Commands 等）が最新と合っているか確認する。更新が必要であれば合わせて提示する
10. `.steering/` 配下のファイル（`architecture.md`, `data-models.md`, `llm-guidelines.md`）を読み込み、変更内容と整合しているか確認する。更新が必要であれば合わせて提示する

## 報告形式

以下の形式で報告する:

- **変更ファイル数**: N件
- **関連する仕様書**: 該当する仕様書のパス一覧（なければ「なし」）
- **更新が必要な箇所**:
  - 仕様書のパス・セクション名・現在の記載・更新案を記載
- **更新不要の場合**: 「仕様書は最新の状態です」と報告
- **新規作成の提案**（関連仕様書が見つからない場合）:
  - 推定した機能名と配置先パス
  - 仕様書の草案
- **パス変更の追従**: リネーム・移動・削除されたファイルの旧パスがドキュメント内に残っていれば、対象ファイルと更新案を記載
- **README.md の更新**: 必要あり/なし（必要な場合は更新案を記載）
- **CLAUDE.md の更新**: 必要あり/なし（必要な場合は更新案を記載）
- **.steering/ の更新**: 必要あり/なし（必要な場合は対象ファイルと更新案を記載）
```
:::

コード変更後にこのスキルを呼び出すと、`docs/specs/` 配下の仕様書やREADME、CLAUDE.mdなど関連ドキュメントとの乖離をチェックし、更新案を提示してくれる。

コードを変更するたびに仕様書やREADMEを手作業で更新するのは面倒で、つい後回しにしがちだ。Skillとして定義しておけば `/update-specs` の一言で済むので、ドキュメントが実装から乖離する心配がなくなった。

### ルールによるワークフロー規約

`.claude/rules/` はClaude Codeのルール機能で、プロジェクト固有の規約やガイドラインをMarkdownファイルとして配置できる。ファイル名やフロントマターの `globs` パターンで適用範囲を制御でき、該当ファイルの編集時に自動的にルールが読み込まれる仕組みだ。

https://code.claude.com/docs/ja/memory#claude%2Frules%2F-%E3%81%A7%E3%83%AB%E3%83%BC%E3%83%AB%E3%82%92%E6%95%B4%E7%90%86%E3%81%99%E3%82%8B

本プロジェクトでは以下の3つのルールを配置し、開発の進め方をClaude Codeに共有している。

- **git-workflow.md**: Issue → feature ブランチ → PR → マージの流れ
- **coding-standards.md**: Python コーディング規約（ruff、型ヒント必須など）
- **implementation-workflow.md**: 実装 → `/update-specs` → テスト → PRの手順

開発中、Claude Codeの公式ドキュメント参照についても指示を入れた。

> **自分**: Skillや仕様ドキュメントの配置、Claudeに関わる実装を進める際は公式ドキュメントを参照して最新の情報を確認するようにしてほしい
>
> **Claude**: 承知しました。今後はスキル・仕様ドキュメント・Claude Code関連の実装時に公式ドキュメントを必ず確認します。
>
> **自分**: メモリだけでなくプロジェクトの実装ワークフローに含めてほしいな

このように、AIとの開発では「どう振る舞ってほしいか」をルールとして明文化し、プロジェクトに組み込むことが重要だと感じた。

:::details git-workflow.md — Git 運用規範
```
---
description: Git 運用規範（daily-tech-news 準拠）
---

# Git 運用規範

## ブランチ戦略

- Issue 作成 → `feature/issue-XX-description` ブランチ → PR → マージ
- main ブランチへの直接コミットは避ける

## PR ルール

- タイトル: 絵文字 + 概要 + `(#Issue番号)`
  - ✨ 新機能 / 🐛 バグ修正 / 📚 ドキュメント / ⚡ パフォーマンス / ♻️ リファクタ
- 本文冒頭: `Closes #番号`

## ラベル

- `enhancement` (#a2eeef): 新機能
- `bug` (#d73a49): バグ修正
- `documentation` (#0075ca): ドキュメント
- `performance` (#0e8a16): 最適化

## コミットメッセージ

- 日本語または英語で簡潔に
- 変更の「何を」「なぜ」が分かるように
```
:::

:::details coding-standards.md — Python コーディング規約
```
---
description: Python コーディング規約
globs: "src/**/*.py,tests/**/*.py"
---

# コーディング規約

## スタイル

- Python 3.11+ の機能を活用（type union `X | Y`、`match` 文など）
- ruff でリント（line-length=100）
- 型ヒントを付ける（Pydantic モデルは必須、ユーティリティ関数も推奨）

## エラーハンドリング

- 記事単位で try-except + 続行（1 記事の失敗でパイプライン全体を止めない）
- 失敗理由を state に記録し、次回再試行可能にする
- ログは `logging_util.py` のロガーを使う

## モジュール構成

- `src/models.py` が全モジュール共通の型定義層
- 依存の方向は上位（main.py）→ 下位（utils/）。循環依存禁止
- LLM プロバイダーは Protocol + Factory パターンで抽象化
```
:::

:::details implementation-workflow.md — 実装ワークフロー
```
---
description: 実装ワークフローのルール
---

# 実装ワークフロー

## ステップごとの進め方

1. Issue を作成する
2. `feature/issue-XX-description` ブランチを切る
3. 実装する
4. `/update-specs` で仕様書との乖離をチェックし、必要なら更新する
5. Test plan の項目をローカルで実行し、動作確認する
6. コミット・プッシュ → PR 作成（確認済みの結果を PR に記載）
7. マージ後、main に戻って次のステップへ

## Claude Code 関連の作業時

Skill（`.claude/skills/`）、ルール（`.claude/rules/`）、CLAUDE.md、仕様ドキュメントの配置など、Claude Code に関わるファイルを作成・変更する際は、**公式ドキュメントを参照して最新の情報を確認してから作業すること**。

Claude Code の仕様（ファイル配置パス、frontmatter 形式など）は変更される可能性があるため、古い知識で進めない。
```
:::

### .steering/ — アーキテクチャ・設計方針の共有

`.steering/` はClaude Code公式ドキュメントに記載のある機能ではなく、Kiroの仕様書駆動開発プロセスや『実践Claude Code入門』第4章「達人に学ぶスペック駆動開発」の内容を参考に取り入れたものだ。

https://gihyo.jp/book/2026/978-4-297-15354-0

CLAUDE.mdの `References` で参照している `.steering/` ディレクトリには、プロジェクトの設計判断やデータモデルの概要をまとめたドキュメントを配置している。

| ファイル | 内容 |
|---|---|
| architecture.md | 処理パイプライン、モジュール依存関係、設計判断の理由 |
| data-models.md | 記事の状態遷移、スキップ理由、主要モデルの一覧 |
| llm-guidelines.md | プロバイダー切り替え、プロンプト設計方針、出力仕様 |

:::details architecture.md — アーキテクチャ概要
```markdown
# アーキテクチャ概要

## 処理パイプライン

Raindrop API → fetch → diff check → fetch URL → extract body → LLM summarize → store JSON → build HTML

## モジュール依存関係

main.py (CLI オーケストレーション)
  ├── config.py          (環境変数)
  ├── logging_util.py    (ロガー)
  ├── raindrop_client.py (API 取得)
  ├── state_store.py     (差分管理)
  ├── content_fetcher.py (HTML 取得)
  ├── content_extractor.py (本文抽出)
  ├── summarizer.py      (LLM 要約)
  ├── article_repository.py (JSON 保存)
  └── html_builder.py    (HTML 生成)

## 設計判断

| 項目 | 判断 | 理由 |
|---|---|---|
| 同期/非同期 | 同期 | MVP は 30 件上限。まず動くものを優先 |
| LLM 抽象化 | Protocol + Factory | SDK の違いを各 Provider 内に閉じ込め |
| JSON 保存 | 1 記事 = 1 ファイル | reprocess が容易 |
| HTML 出力先 | docs/ | GitHub Pages 対応 |
| プロンプト | 外部ファイル prompts/*.txt | コード変更なしで調整可能 |
```
:::

:::details data-models.md — データモデル概要
```markdown
# データモデル概要

## 記事の状態遷移

pending → fetched → extracted → summarized
                  ↘ fallback_ready ↗
          ↘ skipped (理由付き)
          ↘ failed (エラーログ付き)

## スキップ理由

- fetch_failed: URL 取得失敗
- extract_failed: 本文抽出失敗
- summary_input_unavailable: 簡易要約用入力も不足
- unsupported_video: 動画コンテンツ
- unsupported_non_html: 非 HTML コンテンツ
- llm_failed: LLM 応答失敗
- too_short: 本文が短すぎる

## 主要モデル

- RaindropItem: Raindrop API レスポンスの内部表現
- ProcessedArticle: 処理済み記事の正本（data/articles/{id}.json）
- SummaryResult: LLM 出力の JSON スキーマ
- StateEntry / StateIndex: 差分管理用（state/index.json）
```
:::

:::details llm-guidelines.md — LLM プロバイダーガイドライン
```markdown
# LLM プロバイダーガイドライン

## プロバイダー切り替え

- LLM_PROVIDER 環境変数で openai / gemini / anthropic を指定
- LLMProvider Protocol を実装した各プロバイダークラスで抽象化
- Factory 関数で config に基づきインスタンス生成

## プロンプト設計方針

- プロンプトは prompts/ ディレクトリに外部ファイルとして管理
- summarize_full.txt: 本文全文がある場合
- summarize_fallback.txt: メタデータのみの場合
- テンプレート変数: {title}, {url}, {domain}, {text} 等を str.format() で埋め込み

## 出力仕様

- 常に日本語で出力
- JSON 形式を要求し、パース + Pydantic でバリデーション
- 出力スキーマ: topic, summary_3lines, priority, read_now_reason, defer_reason, drop_candidate, drop_reason, keywords

## 本文長による要約モード

- 1500 文字以上: 通常要約（summarize_full.txt）
- 500〜1499 文字: 短文要約（同プロンプト、短め指示）
- 500 文字未満: 簡易要約寄り
- 取得不能: fallback プロンプト（summarize_fallback.txt）
```
:::

これらは `docs/specs/` の詳細仕様書とは別に、Claude Codeが設計判断を素早く参照できるよう要点をまとめたものだ。仕様書は「何を作るか」の定義、`.steering/` は「なぜそう作ったか」の判断根拠という位置づけで使い分けている。

## コスト面の配慮

個人プロジェクトなので、ランニングコストはできるだけ抑えたかった。

| 項目 | 選択 | コスト |
|---|---|---|
| Raindrop.io API | テストトークン | 無料 |
| LLM | Google Gemini（gemini-2.5-flash） | ほぼ無料〜微小 |
| ホスティング | GitHub Pages | 無料 |
| CI/CD | GitHub Actions | 無料枠内 |

LLMプロバイダーはOpenAI・Anthropicにも対応しているが、運用コストを最小限にするためGeminiを採用した。Factoryパターンで切り替え可能な設計にしており、環境変数の変更だけでプロバイダーを差し替えられる。

APIキーの取得についてもClaude Codeに相談しながら進めた。

> **自分**: LLM API キーの料金体系など確認したい
>
> **Claude**: 各プロバイダーの要約向け軽量モデルの料金を比較します。
>
> | プロバイダー | モデル | 入力 (1M tokens) | 出力 (1M tokens) |
> |---|---|---|---|
> | OpenAI | gpt-4.1-mini | $0.40 | $1.60 |
> | Google | gemini-2.0-flash | $0.10 | $0.40 |
> | Anthropic | claude-haiku-4-5 | $0.80 | $4.00 |
>
> コスパ重視なら **Gemini 2.0 Flash** か **gpt-4.1-mini** が適しています。Gemini API は無料枠あり（レート制限付き）で、このプロジェクトの用途（30件/回）なら無料枠で十分です。

> **自分**: Google AI StudioでAPIキーを作成したい。API キー作成時にプロジェクト選択をする必要があるようだけど、このあたりがよくわからない
>
> **Claude**: Google AI Studio での API キー作成は、Google Cloud プロジェクトと紐づく仕組みです。既存プロジェクトがなければ「Create API key in new project」を選択すれば、自動で Google Cloud プロジェクトが作成され、API キーが発行されます。Google Cloud の課金設定（クレジットカード登録）は不要で無料枠から使えます。

料金体系やAPIキーの取得手順まで、開発の流れの中でそのまま相談できるのは便利だった。

また、1回の実行あたりの要約件数に上限を設けたり（Configの `MAX_SUMMARIZE_PER_RUN`）、GitHub Actionsの実行頻度を3日おきにするなど、意図しないコスト増を防ぐ仕組みも取り入れている。

## 要約と優先度判定 — プロンプト設計

このツールの核となるのは、AIによる要約と優先度判定だ。noteにも書いたが、実際に見え方を少し変えるだけで「読んでみよう」という気になる。3行要約と優先度があるだけで、溜まった記事への向き合い方がまったく変わった。

プロンプトは `prompts/` ディレクトリに外部ファイルとして配置し、コード変更なしで調整できるようにしている。

:::details summarize_full.txt — 本文あり要約プロンプト
```
あなたは記事要約アシスタントです。以下の記事を日本語で要約してください。

## 記事情報

- タイトル: {title}
- URL: {url}
- ドメイン: {domain}

## 本文

{text}

## 指示

以下の JSON 形式で出力してください。JSON 以外は出力しないでください。

- topic: 記事の主題を1行で
- summary_3lines: 3行の要約（各行は簡潔に。感想文ではなく内容紹介。事実不明な断定を避ける）
- priority: "high" / "medium" / "low"
  - high: 新規性が高い、関心領域と近い、本文を読む価値がある
  - medium: 要点把握で十分、時間があれば読む価値がある
  - low: 重複度が高い、関心軸から遠い、速報性が過ぎている
- read_now_reason: 今読む価値がある理由（1文）
- defer_reason: 後回しでよい理由（1文）
- drop_candidate: true/false（読む価値が薄い場合 true）
- drop_reason: ドロップ候補の理由（該当しなければ空文字）
- keywords: 記事のキーワード3〜5個
```
:::

:::details summarize_fallback.txt — メタデータのみ簡易要約プロンプト
```
あなたは記事要約アシスタントです。以下の記事のメタデータから、日本語で簡易要約を生成してください。

**注意: 本文は取得できていません。メタデータのみに基づく簡易要約です。**

## メタデータ

{metadata}

## 指示

以下の JSON 形式で出力してください。JSON 以外は出力しないでください。
本文未取得のため、断定的な表現を避け、メタデータから読み取れる範囲で要約してください。
```
:::

ポイントは以下の点だ。

- **JSON固定出力**: LLMのレスポンスをPydanticでバリデーションするため、出力形式をJSON固定にしている
- **「感想文ではなく内容紹介」**: プロンプトに明示しないと、AIは「この記事は素晴らしい」のような感想を返しがち。事実ベースの要約に絞る指示が重要
- **優先度の判定基準**: high / medium / low の基準を具体的に定義。「新規性が高い」「関心軸から遠い」など、判断の軸を明示している
- **read_now_reason と defer_reason の両方を出力**: 「今読む理由」と「後回しにしてよい理由」の両面を出すことで、判断材料として機能する
- **フォールバック対応**: 本文が取得できなかった場合はメタデータのみの簡易要約に切り替え。「断定的な表現を避ける」よう指示している

### 実際の要約結果

実際にAIが生成した要約の例を紹介する。

各記事には「今読む理由（read_now_reason）」と「後回しでよい理由（defer_reason）」の両方が出力される。highの記事には説得力のある「今読む理由」が添えられ、mediumやlowの記事には具体的な「後回しでよい理由」が示される。

**high判定の例:**

> **topic**: DeNA南場智子氏が語るAI時代の変革と未来予測
> **summary_3lines**:
> - DeNAのAI変革による開発生産性の劇的な向上
> - 「エンバイロメントエンジニアリング」がAIエージェント時代の鍵
> - スピードと専門特化（〇〇×AI）が今後のプロダクトの命運を握る
> **read_now_reason**: ClaudeのAgent Skillsを利用している場合、スキルの品質保証とメンテナンスに直結する重要な新機能であるため

**medium判定の例:**

> **topic**: 予測市場についての概要、仕組み紹介
> **defer_reason**: 本書籍は多岐にわたる内容を網羅しているため、深い理解を得るにはまとまった読書時間が必要となります。

high判定でも`defer_reason`が空になることがあるが、これはAIが「後回しにする理由が見当たらない」と判断したケースだ。ダッシュボード上では折りたたみで判定理由を確認でき、記事を開く前に読むべきかを判断できる。

![](https://storage.googleapis.com/zenn-user-upload/d72996aa56ac-20260328.png)

## プロトタイプからv1.0.0リリースまでの記録

開発はStep 0（基盤整備）からStep 8（テスト）まで段階的に進め、各ステップでIssue → ブランチ → PR → マージのサイクルを回した。

> **自分**: ステップごとにコミット、PR作成する形にしたい。段階的に実装を進めたいので

この指示により、各Stepが独立したPRとしてマージされ、変更の追跡がしやすくなった。

### Step 0: 仕様書・プロジェクト基盤の整備

ChatGPTで用意した実装計画をClaude Codeに渡し、仕様駆動開発に則って8つの仕様ドキュメントを `docs/specs/` に配置。あわせて `pyproject.toml`、`.env.example` などプロジェクト基盤の初期化を実施した。

| ファイル | 内容 |
|---|---|
| 01_overview.md | プロジェクト目的・スコープ |
| 02_data_models.md | データモデル・状態遷移 |
| 03_api_integration.md | Raindrop.io API連携 |
| 04_content_extraction.md | 本文取得・抽出 |
| 05_summarization.md | AI要約仕様 |
| 06_html_output.md | HTML出力 |
| 07_state_management.md | 状態管理・差分処理 |
| 08_cli_interface.md | CLIコマンド仕様 |

各仕様書の全容は以下の折りたたみで確認できる。

:::details 01_overview.md — プロジェクト概要
# 01. プロジェクト概要

## 目的

Raindrop.io に溜めた「あとで読む」記事を、全文を読む前に一次判定できる状態にする。

- 今読むべきか
- 後回しでよいか
- 読まずに捨ててよいか

## スコープ（初期リリース）

### 対象

- Raindrop.io の単一コレクション
- 定期取得または手動実行
- URL 先本文の抽出
- AI による日本語要約
- HTML 一覧出力
- JSON による永続化と差分管理

### 対象外

- 動画コンテンツの要約（判別のみ対応）
- 複数コレクション横断処理
- DB 導入
- Web UI からの編集
- note への自動投稿
- Playwright などの重いブラウザ自動化

## 固定条件

- 対象コレクションは 1 つ
- 本文抽出失敗時は簡易要約を試みる
- 簡易要約も不可ならスキップ
- 優先度は 3 段階（high / medium / low）
- 1 回の要約件数上限は 30 件
- 出力の主用途は HTML
- 要約出力は日本語（英語記事も日本語で要約）
- 動画は初期リリースの対象外（判別はする）

## 用語定義

| 用語 | 説明 |
|---|---|
| Raindrop | Raindrop.io に保存された 1 件のブックマーク |
| コレクション | Raindrop.io 上のブックマークフォルダ |
| 本文抽出 | URL 先の HTML から記事本文テキストを取り出すこと |
| 簡易要約 | 本文が取れない場合にタイトル・excerpt 等から生成する要約 |
| 正本 | データの権威あるソース。本プロジェクトでは `data/articles/{id}.json` |
| 差分管理 | `state/index.json` で既処理記事を追跡し、新規のみ処理すること |
:::

:::details 02_data_models.md — データモデル定義
# 02. データモデル定義

## 状態 Enum

### ArticleState

```
pending → fetched → extracted → summarized
                  ↘ fallback_ready ↗
          ↘ skipped
          ↘ failed
```

- `pending`: 未処理
- `fetched`: HTML 取得済み
- `extracted`: 本文抽出済み
- `fallback_ready`: 簡易要約用入力構成済み
- `summarized`: 要約完了
- `skipped`: スキップ（理由付き）
- `failed`: 処理失敗（エラーログ付き）

### SkipReason

`fetch_failed` / `extract_failed` / `summary_input_unavailable` / `unsupported_video` / `unsupported_non_html` / `llm_failed` / `too_short`

### ContentType

`article` / `video` / `other`

### Priority

`high` / `medium` / `low`

### ManualStatus

`untriaged` / `read` / `keep` / `drop`

## RaindropItem

Raindrop API レスポンスから抽出する内部モデル。

```json
{
  "raindrop_id": 123456789,
  "collection_id": 111111,
  "title": "Example title",
  "url": "https://example.com/article",
  "domain": "example.com",
  "created_at": "2026-03-25T00:00:00Z",
  "tags": [],
  "excerpt": "short excerpt",
  "type": "link",
  "cover": "",
  "note": ""
}
```

## SummaryResult

LLM 出力の JSON スキーマ。

```json
{
  "topic": "主題",
  "summary_3lines": ["1行目", "2行目", "3行目"],
  "priority": "high",
  "read_now_reason": "今読む価値の理由",
  "defer_reason": "後回しでよい理由",
  "drop_candidate": false,
  "drop_reason": "",
  "keywords": ["キーワード1", "キーワード2", "キーワード3"]
}
```

## ProcessedArticle

処理済み記事の正本データ。`data/articles/{raindrop_id}.json` に保存。

```json
{
  "raindrop_id": 123456789,
  "collection_id": 111111,
  "title": "記事タイトル",
  "url": "https://example.com/article",
  "domain": "example.com",
  "created_at": "2026-03-25T00:00:00Z",
  "fetched_at": "2026-03-25T08:30:00Z",
  "source_language": "en",
  "output_language": "ja",
  "content_type": "article",
  "content_status": "ok",
  "fetch_status": "ok",
  "extract_method": "trafilatura",
  "content_chars": 6842,
  "content_hash": "sha256:...",
  "summary_input_type": "fulltext",
  "topic": "AI と検索体験の変化",
  "summary_3lines": [
    "この記事は〜について述べている。",
    "主な論点は〜である。",
    "実務上は〜という示唆がある。"
  ],
  "priority": "medium",
  "read_now_reason": "最近の関心領域と近く、今後の発信や調査に活かしやすいため。",
  "defer_reason": "要点把握だけでも十分で、緊急性は高くないため。",
  "drop_candidate": false,
  "drop_reason": "",
  "keywords": ["AI", "検索", "プロダクト"],
  "model_provider": "openai",
  "model_name": "gpt-4.1-mini",
  "summarized_at": "2026-03-25T08:31:00Z",
  "manual_status": "untriaged",
  "notes": ""
}
```

## StateIndex

差分管理用。`state/index.json` に保存。

```json
{
  "last_run_at": "2026-03-25T08:31:00Z",
  "items": {
    "123456789": {
      "status": "summarized",
      "content_hash": "sha256:...",
      "summarized_at": "2026-03-25T08:31:00Z"
    },
    "987654321": {
      "status": "skipped",
      "reason": "summary_input_unavailable",
      "updated_at": "2026-03-25T08:31:00Z"
    }
  }
}
```
:::

:::details 03_api_integration.md — Raindrop.io API 連携仕様
# 03. Raindrop.io API 連携仕様

## 対象

単一コレクションのみ。

## 認証

- テストトークンを使用（OAuth 不要）
- ヘッダー: `Authorization: Bearer {RAINDROP_TOKEN}`

## エンドポイント

```
GET https://api.raindrop.io/rest/v1/raindrops/{collectionId}
```

### パラメータ

| パラメータ | 値 | 説明 |
|---|---|---|
| `perpage` | `50` | API 最大値 |
| `sort` | `-created` | 新しい順 |
| `page` | `0, 1, 2...` | ページネーション |

## ページネーション

- `page=0` から開始
- レスポンスの `items` が空になるまでループ
- 差分処理により通常は 1〜2 ページで十分

## レスポンスマッピング

API レスポンスの各フィールドを `RaindropItem` にマッピングする。

| API フィールド | 内部フィールド |
|---|---|
| `_id` | `raindrop_id` |
| `collection.$id` | `collection_id` |
| `title` | `title` |
| `link` | `url` |
| `domain` | `domain` |
| `created` | `created_at` |
| `tags` | `tags` |
| `excerpt` | `excerpt` |
| `type` | `type` |
| `cover` | `cover` |
| `note` | `note` |

## 生データ保存

取得した生レスポンスを `data/raw/latest_raindrops.json` に保存する。
デバッグおよび再処理時の参照用。

## エラーハンドリング

- 401: トークン無効 → エラーメッセージを出して終了
- 429: レートリミット → ログ出力して終了
- 5xx: サーバーエラー → ログ出力して終了
- ネットワークエラー → ログ出力して終了
:::

:::details 04_content_extraction.md — 本文取得・抽出仕様
# 04. 本文取得・抽出仕様

## フォールバックチェーン

```
1. HTTP GET で HTML 取得
2. trafilatura で本文抽出
3. readability-lxml でフォールバック
4. タイトル / excerpt / URL / domain / tags で簡易要約入力を構成
5. すべて失敗 → スキップ
```

## HTTP 取得

- ライブラリ: httpx（同期クライアント）
- タイムアウト: `REQUEST_TIMEOUT_SECONDS`（デフォルト 20 秒）
- User-Agent: `USER_AGENT` 環境変数
- リダイレクト: 自動追跡
- 動画 URL は HTTP 取得前にスキップ判定

## 本文抽出

### trafilatura（優先）

```python
trafilatura.extract(html, include_comments=False, include_tables=True)
```

成功時: `extract_method = "trafilatura"`

### readability-lxml（フォールバック）

trafilatura が失敗、または結果が 100 文字未満の場合に使用。

```python
from readability import Document
doc = Document(html)
doc.summary()  # HTML → テキスト変換が必要
```

成功時: `extract_method = "readability"`

## 簡易要約用入力

本文が取れない場合、以下を AI に渡す。

- title
- excerpt
- URL
- domain
- tags
- 取得日時
- og:description（取得可能な場合）

`summary_input_type = "metadata"`

## 本文長ルール

| 文字数 | 扱い |
|---|---|
| 1500 文字以上 | 通常要約 |
| 500〜1499 文字 | 短文要約 |
| 500 文字未満 | 簡易要約寄り |
| 取得不能 | fallback or skip |

## 動画コンテンツの判定

### 判定方法

1. Raindrop API の `type == "video"` フィールド
2. ドメインベース判定:
   - `youtube.com`, `youtu.be`
   - `vimeo.com`
   - `nicovideo.jp`, `nico.ms`
   - `dailymotion.com`
   - `twitch.tv`

### 処理

- `content_type = "video"`
- `status = "skipped"`
- `reason = "unsupported_video"`
- HTML 出力では動画であることを表示

## og:description の取得

HTML 取得時に `<meta property="og:description">` を抽出し、簡易要約用入力に含める。
BeautifulSoup ではなく、正規表現または trafilatura の metadata 機能を使用。
:::

:::details 05_summarization.md — AI 要約仕様
# 05. AI 要約仕様

## 出力言語

- 常に日本語で出力
- 英語記事も日本語で要約
- 元言語は `source_language` に保存

## LLM プロバイダー抽象化

### Protocol

```python
class LLMProvider(Protocol):
    def generate(self, prompt: str) -> str: ...
```

### 実装

- `OpenAIProvider`: openai SDK
- `GeminiProvider`: google-generativeai SDK
- `AnthropicProvider`: anthropic SDK

### Factory

```python
def create_provider(config: Config) -> LLMProvider:
    match config.llm_provider:
        case "openai": return OpenAIProvider(config)
        case "gemini": return GeminiProvider(config)
        case "anthropic": return AnthropicProvider(config)
```

## プロンプト

### summarize_full.txt

本文全文がある場合に使用。以下を要求:
- topic: 主題
- summary_3lines: 3 行要約（簡潔に、感想文にしない、事実ベース）
- priority: high / medium / low
- read_now_reason: 今読む価値の理由
- defer_reason: 後回しでよい理由
- drop_candidate: boolean
- drop_reason: ドロップ候補の理由
- keywords: キーワード 3〜5 個

### summarize_fallback.txt

メタデータのみの場合に使用。「本文未取得前提」であることを明示。

## 出力フォーマット

JSON 固定。Pydantic の `SummaryResult` でバリデーション。

## 要約ルール

- 3 行要約は簡潔に（1 行あたり長くなりすぎない）
- 記事の内容紹介であり、感想文にしない
- 日本語として自然に整える
- 英語記事でも直訳調にしすぎない
- 事実不明な断定を避ける

## 優先度判定基準

### high（今読む価値が高い）

- 新規性が高い
- 関心領域と近い
- 今後の発信や調査に活かせる
- 実務や制作に影響がある
- 要約だけでは足りず、本文を読む価値がある

### medium（時間があれば読む価値がある）

- 要点は短く把握できる
- 深掘り前提でまとまった時間が必要

### low（要約把握で十分、後回しでよい）

- 内容の重複度が高そう
- 関心軸から遠い
- 速報性が過ぎて後追い価値が低い

## エラーハンドリング

- LLM 応答が JSON でない場合 → リトライ 1 回
- リトライ失敗 → `status = "failed"`, `reason = "llm_failed"`
- タイムアウト → 同上
:::

:::details 06_html_output.md — HTML 出力仕様
# 06. HTML 出力仕様

## 目的

一覧を見返しながら、読む / 保留 / 捨てる判断をしやすくすること。

## 出力先

`docs/index.html`（GitHub Pages 対応）

## テンプレートエンジン

Jinja2

## 記事カードの必須項目

- タイトル（元記事 URL へのリンク）
- ドメイン
- 追加日（created_at）
- 3 行要約
- 主題（topic）
- 優先度バッジ（high / medium / low）
- 今読む理由（read_now_reason）
- 後回し理由（defer_reason）
- ドロップ候補かどうか
- 本文取得ステータス（content_status）
- 手動ステータス（manual_status）
- メモ欄（notes）

## ソート順

初期表示:
1. priority（high → medium → low）
2. created_at の新しい順

## フィルタ

- JS なしでも全記事が読める HTML にする
- `docs/app.js` で軽いクライアントサイド絞り込みを追加
  - 優先度フィルタ
  - ステータスフィルタ
  - キーワード検索

## スタイリング

`docs/styles.css`:
- 優先度ごとの色分け（high=赤系 / medium=黄系 / low=灰系）
- レスポンシブ対応
- カード形式レイアウト
- ドロップ候補は視覚的に薄く表示

## スキップ記事の表示

- 要約済み記事とは別セクションに表示
- スキップ理由を明示
- 動画は動画アイコンで区別

## 統計情報

HTML のヘッダーに以下を表示:
- 総記事数
- 要約済み件数
- スキップ件数
- 最終実行日時
:::

:::details 07_state_management.md — 状態管理仕様
# 07. 状態管理仕様

## 概要

`state/index.json` で記事の処理状態を管理し、差分処理を実現する。

## 差分判定ルール

### 基本ルール

- `raindrop_id` を主キーとする
- `summarized` 済みは通常スキップ
- `content_hash` が変わった場合は再要約対象にできる設計にする
- 初期リリースでは再要約は明示的フラグ（`reprocess` コマンド）のみ

### 初期リリースの実運用ルール

- 新規記事のみ要約
- 1 回の処理上限は 30 件（`MAX_SUMMARIZE_PER_RUN`）
- 上限を超えた分は次回へ持ち越し

## 状態遷移

```
[Raindrop API取得]
    ↓
  pending ──→ fetched ──→ extracted ──→ summarized
    │            │           │
    │            │           └→ fallback_ready → summarized
    │            │
    │            └→ skipped (fetch_failed)
    │            └→ failed
    │
    └→ skipped (unsupported_video, unsupported_non_html)
```

## index.json 構造

```json
{
  "last_run_at": "ISO8601",
  "items": {
    "{raindrop_id}": {
      "status": "summarized | skipped | failed | pending",
      "content_hash": "sha256:... | null",
      "reason": "スキップ理由 | null",
      "summarized_at": "ISO8601 | null",
      "updated_at": "ISO8601"
    }
  }
}
```

## 操作

### 新規記事の検出

```
Raindrop API の ID 一覧 - index.json の既存 ID = 新規記事
```

### 処理対象の決定

1. 新規記事を `created_at` の新しい順にソート
2. 先頭から `MAX_SUMMARIZE_PER_RUN` 件を処理対象にする
3. 残りは `pending` として index に追加

### 再処理

- `reprocess --id`: 指定 ID のエントリを index から削除して再処理
- `reprocess-failed`: `status == "failed"` のエントリを一括で再処理対象にする
:::

:::details 08_cli_interface.md — CLI インターフェース仕様
# 08. CLI インターフェース仕様

## 実行方法

```bash
python -m src.main <command>
```

## コマンド一覧

### `run`

フルパイプラインを実行。

```bash
python -m src.main run
```

処理フロー:
1. Config ロード
2. StateIndex 読込
3. Raindrop API から記事取得
4. 差分検出（最大 30 件）
5. 各記事: fetch → extract → summarize
6. StateIndex 更新
7. HTML 生成
8. 完了サマリー表示

### `fetch-only`

Raindrop 取得と生データ保存のみ。

```bash
python -m src.main fetch-only
```

### `build-html`

保存済み JSON から HTML を再生成。

```bash
python -m src.main build-html
```

### `reprocess`

特定記事を再処理。

```bash
python -m src.main reprocess --id 123456789
```

### `reprocess-failed`

失敗・スキップ記事を一括再試行。

```bash
python -m src.main reprocess-failed
```

## 共通オプション

| オプション | 説明 | デフォルト |
|---|---|---|
| `--dry-run` | 実際の処理は行わず、対象記事を表示 | false |
| `--verbose` | 詳細ログ出力 | false |

## 出力

- rich ライブラリで見やすい CLI 出力
- 処理サマリー: 取得件数、新規件数、要約件数、スキップ件数、失敗件数
- プログレスバー: 記事処理の進捗表示
:::

初期化後のプロジェクト構成は以下の通り。

```
tsuyu-mi/
├── .claude/                    # Claude Code 設定
│   ├── rules/                  # 開発ワークフロー規約
│   └── skills/                 # スキル定義
├── .github/workflows/          # GitHub Actions
├── data/
│   ├── articles/               # 処理済み記事 JSON
│   └── raw/                    # Raindrop API 生データ（非公開）
├── docs/
│   ├── index.html              # ダッシュボード（GitHub Pages）
│   └── specs/                  # 仕様書
├── prompts/                    # LLM プロンプトテンプレート
├── src/                        # Python ソースコード
├── tests/                      # テストコード
├── state/                      # 状態管理（index.json）
└── pyproject.toml              # プロジェクト設定
```

### Step 1: config + logging + models（基盤層）

全モジュール共通の基盤となる3ファイルを実装した。

**config.py** — `pydantic-settings` の `BaseSettings` を使い、`.env` ファイルから環境変数を型安全に読み込む。バリデーションメソッドで必須項目の未設定を検出する設計。

```python
class Config(BaseSettings):
    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}

    raindrop_token: str = ""
    raindrop_collection_id: int = 0
    llm_provider: str = "openai"
    llm_api_key: str = ""
    llm_model: str = ""
    max_summarize_per_run: int = 10
    request_timeout_seconds: int = 20
```

**logging_util.py** — `rich` ライブラリの `RichHandler` でログ出力を見やすくしている。多重初期化を防ぐハンドラー存在チェック付き。

```python
def setup_logger(level: str = "INFO") -> logging.Logger:
    logger = logging.getLogger("raindrop_summarizer")
    if logger.handlers:
        return logger
    handler = RichHandler(rich_tracebacks=True, show_time=True, show_path=False)
    handler.setFormatter(logging.Formatter("%(message)s"))
    logger.addHandler(handler)
    logger.setLevel(getattr(logging, level.upper(), logging.INFO))
    return logger
```

**models.py** — Pydanticの `BaseModel` と `str, Enum` の組み合わせで、記事の状態遷移や処理結果を型安全に定義。`ArticleState` は7つの状態を持ち、パイプライン全体の流れを表現する。

```python
class ArticleState(str, Enum):
    pending = "pending"
    fetched = "fetched"
    extracted = "extracted"
    fallback_ready = "fallback_ready"
    summarized = "summarized"
    skipped = "skipped"
    failed = "failed"
```

### Step 2: state_store + article_repository（永続化層）

差分処理と記事データの永続化を担う2ファイルを実装した。

**state_store.py** — `state/index.json` で記事の処理状態を一元管理する。Raindrop IDをキーとした辞書形式で、新規記事の検出や処理上限の制御を行う。

```python
class StateStore:
    def get_new_articles(
        self, raindrops: list[RaindropItem], max_count: int = 30
    ) -> list[RaindropItem]:
        """未処理の新規記事を抽出する。created_at の新しい順、上限 max_count 件。"""
        existing_ids = set(self.index.items.keys())
        new_items = [r for r in raindrops if str(r.raindrop_id) not in existing_ids]
        new_items.sort(key=lambda r: r.created_at, reverse=True)
        # 上限を超えた分は pending として登録
        for item in new_items[max_count:]:
            self._set_entry(str(item.raindrop_id), ArticleState.pending)
        return new_items[:max_count]
```

**article_repository.py** — 1記事=1JSONファイル（`data/articles/{raindrop_id}.json`）の設計で、Pydanticの `model_dump(mode="json")` による安全なシリアライズを行う。個別ファイルの読み込み失敗はスキップする耐障害設計。

```python
class ArticleRepository:
    def save(self, article: ProcessedArticle) -> Path:
        """記事を JSON ファイルとして保存する。"""
        self.articles_dir.mkdir(parents=True, exist_ok=True)
        path = self.articles_dir / f"{article.raindrop_id}.json"
        data = article.model_dump(mode="json")
        path.write_text(
            json.dumps(data, ensure_ascii=False, indent=2), encoding="utf-8"
        )
        return path
```

この1記事=1ファイルの設計により、特定記事の再処理やデバッグが容易になっている。

### Step 3: raindrop_client（API取得層）

Raindrop.io APIからコレクションのブックマーク一覧を取得するクライアント。httpxを使い、ページネーション対応で全件取得する。

```python
class RaindropClient:
    def fetch_collection(self) -> list[RaindropItem]:
        """コレクションの全記事を取得する。ページネーション対応。"""
        all_items: list[dict] = []
        page = 0
        while True:
            url = f"{API_BASE}/raindrops/{collection_id}"
            params = {"perpage": 50, "sort": "-created", "page": page}
            response = self.client.get(url, params=params)
            items = response.json().get("items", [])
            if not items:
                break
            all_items.extend(items)
            page += 1
        return [self._map_item(item) for item in all_items]
```

### Step 4: content_fetcher + content_extractor（本文取得層）

記事URLからHTML本文を取得・抽出するモジュール。trafilatura → readability-lxml → メタデータフォールバックの3段構えで、抽出失敗時も簡易要約で対応する。

```python
def extract_body(html: str, raindrop: RaindropItem, og_description: str = "") -> ExtractionResult:
    """HTML から本文を抽出する。フォールバックチェーン付き。"""

    # 1. trafilatura
    text = trafilatura.extract(html, include_comments=False, include_tables=True)
    if text and count_chars(text) >= 100:
        return ExtractionResult(text=clean_text(text), method="trafilatura", ok=True)

    # 2. readability-lxml
    doc = Document(html)
    text = trafilatura.extract(doc.summary()) or ""
    if count_chars(text) >= 100:
        return ExtractionResult(text=clean_text(text), method="readability", ok=True)

    # 3. メタデータフォールバック
    fallback = _build_fallback_input(raindrop, og_description)
    if fallback:
        return ExtractionResult(method="metadata", ok=True, fallback_input=fallback)

    return ExtractionResult()  # すべて失敗
```

### Step 5: summarizer（AI要約層）

LLMプロバイダーを抽象化し、Protocol + Factoryパターンで3プロバイダー（OpenAI / Gemini / Anthropic）に対応。プロンプトは `prompts/*.txt` に外部化し、コード変更なしで調整可能にしている。

```python
class LLMProvider(Protocol):
    def generate(self, prompt: str) -> str: ...

def create_provider(config: Config) -> LLMProvider:
    match config.llm_provider:
        case "openai": return OpenAIProvider(config)
        case "gemini": return GeminiProvider(config)
        case "anthropic": return AnthropicProvider(config)
```

LLMのレスポンスはJSON固定で、PydanticのSummaryResultでバリデーション。レスポンスがJSONでない場合は5秒待機後にリトライする。

### Step 6: html_builder（出力層）

Jinja2テンプレートで優先度付きHTMLダッシュボードを生成。`docs/index.html` に出力し、GitHub Pages対応。優先度ごとの色分け、フィルタ機能、レスポンシブ対応を備える。

### Step 7: main.py CLI統合

Click + Richで各モジュールをCLIとして統合。処理パイプライン全体を通して実行できる。

```
Raindrop API → fetch → diff check → fetch URL → extract body → LLM summarize → store JSON → build HTML
```

| コマンド | 説明 |
|---|---|
| `run` | フルパイプライン実行 |
| `fetch-only` | Raindrop取得と生データ保存のみ |
| `build-html` | 保存済みJSONからHTML再生成 |
| `reprocess --id` | 特定記事を再処理 |
| `reprocess-failed` | 失敗記事を一括再試行 |

### Step 8: ユニットテスト

pytest + respxでユニットテストを実装。外部API呼び出しはモックし、各モジュールの単体テストを網羅している。

```python
class MockProvider:
    """テスト用のモック LLM プロバイダー。"""
    def __init__(self, response: str = ""):
        self.response = response
    def generate(self, prompt: str) -> str:
        return self.response

class TestSummarizeFulltext:
    def test_success(self):
        mock = MockProvider(_valid_response())
        result = summarize_fulltext(mock, "本文テキスト", title="タイトル")
        assert result is not None
        assert result.topic == "テスト主題"

    def test_failure_returns_none(self):
        result = summarize_fulltext(FailingProvider(), "本文")
        assert result is None
```

本文抽出のテストでは、十分なコンテンツがある場合とフォールバックのケースをそれぞれ検証している。

```python
class TestExtractBody:
    def test_with_enough_content(self):
        html = "<html><body><article>" + "テスト本文。" * 200 + "</article></body></html>"
        result = extract_body(html, item)
        assert result.ok
        assert result.method in ("trafilatura", "readability")

    def test_empty_html_fallback(self):
        result = extract_body("<html><body></body></html>", item_with_title)
        assert result.ok
        assert result.method == "metadata"
```

### Phase 2〜3: 改善フェーズ

MVPの実装が完了した後、Phase 2（運用強化）、Phase 3（自動運用）へと進んだ。

> **自分**: MVPの実装ができたので次フェーズに進みたい

代表的な改善Issue:

| Issue | 内容 |
|---|---|
| [#33](https://github.com/unsolublesugar/tsuyu-mi/issues/33) | 文字化け問題の対応 |
| [#34](https://github.com/unsolublesugar/tsuyu-mi/issues/34) | X(Twitter)記事形式ポスト対応 |
| [#35](https://github.com/unsolublesugar/tsuyu-mi/issues/35) | GitHub Pagesの404問題修正 |
| [#39](https://github.com/unsolublesugar/tsuyu-mi/issues/39) | サービス名をTsuyu-miにリネーム |
| [#47](https://github.com/unsolublesugar/tsuyu-mi/issues/47) | Raindrop.ioフローティングリンクボタン |
| [#53](https://github.com/unsolublesugar/tsuyu-mi/issues/53) | READMEを英語ベースにリライト・日本語版切り替え |
| [#55](https://github.com/unsolublesugar/tsuyu-mi/issues/55) | セキュリティ改善（SHA ピン留め等） |

### サービス名「Tsuyu-mi」の命名

開発当初はリポジトリ名も `raindrop-collection-summarizer` だったが、途中でサービス名を定義した。

https://github.com/unsolublesugar/tsuyu-mi/issues/39

多くの人に使ってもらうことを前提としたサービスではないが、自分のプロジェクトに名前を付けて愛着を持つことは大事だと思っている。名前があると雑に扱えなくなるし、改善を続けるモチベーションにもなる。

> Raindrop → 雫（しずく） → 露（つゆ） → 露見（つゆみ）— 溜まった記事の中身を「露わに見る」という意味を込め「Tsuyu-mi」と名付けている。
> 
> 意味のある名前を付けると、愛着も湧く。

https://note.com/unsoluble_sugar/n/n543659f80c1e

### READMEの英語化・日本語版切り替え

リポジトリ公開にあたり、READMEを英語ベースにリライトした。

![](https://storage.googleapis.com/zenn-user-upload/7e276099c74f-20260328.png)

日本語版は `README_ja.md` として用意し、冒頭に切り替えリンクを配置。使用技術やライセンスのバッジ、GitHub Markdownのアラート表記（NOTE、TIP、WARNING、IMPORTANT）も取り入れている。

![](https://storage.googleapis.com/zenn-user-upload/76e04860f08a-20260328.png)

![](https://storage.googleapis.com/zenn-user-upload/7e0c700dfa4e-20260328.png)

![](https://storage.googleapis.com/zenn-user-upload/e730fdebccd7-20260328.png)

これは他のリポジトリで見かけて「良いな」と思ったので採用してみた。

https://github.com/unsolublesugar/tsuyu-mi/issues/53

### UI改善 — フローティングリンクボタン

要約の一覧を眺めつつ、元のコレクションの状態もすぐ確認できるようにしたかったので、Raindrop.ioへのフローティングリンクボタンを追加した。別プロジェクト（[daily-tech-news](https://github.com/unsolublesugar/daily-tech-news)）のダークモード切り替えボタンを参考に実装している。

https://github.com/unsolublesugar/tsuyu-mi/issues/47

daily-tech-newsのダークモード切り替えボタン:

```css
#theme-toggle {
    position: fixed;
    top: 20px;
    right: 20px;
    width: 48px;
    height: 48px;
    border-radius: 50%;
    background-color: var(--card-bg);
    box-shadow: 0 4px 12px var(--shadow-medium);
    z-index: 1000;
}
```

これを参考に、tsuyu-miでは[Raindrop.io公式サイト](https://raindrop.io)から取得したSVGアイコンを使ったフローティングリンクを実装:

```css
.raindrop-link {
    position: fixed;
    top: 20px;
    right: 20px;
    width: 48px;
    height: 48px;
    border-radius: 50%;
    background: #fff;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
    z-index: 1000;
}
```

プロジェクトをまたいだ参照もスムーズにできるのは、Claude Codeの強みのひとつだ。

## 開発中に起きた問題とアプローチ

### 文字化け問題

ITMedia等のShift_JIS系サイトで本文取得時に文字化けが発生した。HTTPレスポンスのContent-Typeやmetaタグからエンコーディングを判定し、適切にデコードする処理を追加。

https://github.com/unsolublesugar/tsuyu-mi/issues/33

### X（Twitter）ポストの本文取得

Xの記事形式ポストは通常のHTMLスクレイピングでは本文を取得できなかった。vxtwitter API経由で取得する方式で対応。

https://github.com/unsolublesugar/tsuyu-mi/issues/34

### GitHub Pagesの404問題

GitHub Actions実行中にPages デプロイが走ると404になる問題が発生。ワークフローの実行順序（コミット → デプロイ）を見直して解決。

https://github.com/unsolublesugar/tsuyu-mi/issues/35

### Raindrop MCPサーバーの検討

開発途中で、Raindrop.io用のMCPサーバーの存在に気づいた。使えば実装がシンプルになるかもと思い、Claude Codeに相談。

https://zenn.dev/takna/articles/mcp-server-tutorial-10-raindropio

> **自分**: RaindropのMCPサーバーがあるようだけど、これ使ったらもっと実装がシンプルになる？ホスティング前提だと微妙かな
>
> **Claude**: このプロジェクトではMCPサーバーは合わないと思います。MCPはLLMがリアルタイムに外部サービスと対話する仕組み。このツールはPythonバッチスクリプトで、GitHub Actionsでcron実行する前提だと、MCPサーバーを起動・接続する仕組み自体が不要なオーバーヘッドになります。

技術選定の場面でも、Claude Codeが明確な判断を返してくれるのは頼もしかった。

### その他

- 新規記事がない実行時の `UnboundLocalError` → 変数初期化を修正
- スマホ表示でのレイアウト崩れ → レスポンシブCSS調整
- タイムゾーンの不整合 → JST への明示的な変換処理を追加

https://github.com/unsolublesugar/tsuyu-mi/issues/57

https://github.com/unsolublesugar/tsuyu-mi/issues/45

https://github.com/unsolublesugar/tsuyu-mi/pull/32

## セキュリティ面で気をつけたこと

個人プロジェクトとはいえ、公開リポジトリにする前提だったため、セキュリティには注意を払った。

### リポジトリ公開前の精査

Claude Codeにセキュリティ監査を依頼し、プロジェクト全体を精査してもらった。

> **自分**: 本プロジェクトについてセキュリティ的に懸念のある実装や設計が無いか精査してほしい

結果、GitHub Actionsのアクションがタグ指定だったため、SHAピン留めに変更するなどの改善を実施。

https://github.com/unsolublesugar/tsuyu-mi/issues/55

### 機密データの取り扱い

`data/raw/latest_raindrops.json` にはRaindrop APIのレスポンス（ブックマーク一覧）が含まれており、公開前にGit履歴からも完全削除した。

> **自分**: ひと通り実装が済んだのでリポジトリをpublicに切り替えようと思ってる。切り替えに際する懸念点や、公開したらまずそうなデータがないか精査してほしい。
>
> **Claude**: 切り替え前のチェックポイント: 
　`.env` は `.gitignore` 済み ✅
　コード内にハードコードされたシークレットなし ✅ 
　`data/raw/latest_raindrops.json` に実データが含まれている — 公開しても問題ないか？
>
> **自分**: クリティカルなものは `data/raw/latest_raindrops.json` かな。これを秘匿化することは可能？履歴からも完全削除してほしい

`git filter-repo` による履歴書き換え → force push で対応。Claude Codeが一連の操作を案内・実行してくれた。

### その他の対策

- `.claude/settings.local.json` のGit追跡除外
- mainブランチ保護（Ruleset で直接push禁止）
- 依存バージョンの下限固定 + `pip-audit` による脆弱性スキャン
- HTTPリダイレクト上限の明示設定

https://github.com/unsolublesugar/tsuyu-mi/issues/41

上記でピックアップしたもの含め、クローズ済みIssueの一覧は以下で確認できる。

https://github.com/unsolublesugar/tsuyu-mi/issues?q=is%3Aissue%20state%3Aclosed

## GitHub Actionsによる定期実行・デプロイ

セキュリティ面の精査を経てリポジトリを公開した後は、自動化とデプロイの整備に移った。手元でのローカル実行で動作確認はできていたので、次はこれを自動化し、Web上でいつでも確認できる状態にしたかった。

GitHub Actionsで3日おき（JST 7:00）に自動実行し、結果をGitHub Pagesにデプロイしている。手動実行（workflow_dispatch）にも対応。

以下が実際のワークフロー設定だ。

```yaml
name: Summarize & Deploy

on:
  schedule:
    - cron: '0 22 */3 * *'  # UTC 22:00 = JST 7:00、3日おき
  workflow_dispatch:

permissions:
  contents: write
  pages: write
  id-token: write

concurrency:
  group: summarize
  cancel-in-progress: true

jobs:
  summarize:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@34e114876b0b11c390a56381ad16ebd13914f8d5  # v4

      - uses: actions/setup-python@a26af69be951a213d495a4c3e4e4022e16d87065  # v5
        with:
          python-version: '3.11'

      - run: pip install -e .

      - name: Run summarizer
        env:
          RAINDROP_TOKEN: ${{ secrets.RAINDROP_TOKEN }}
          RAINDROP_COLLECTION_ID: ${{ secrets.RAINDROP_COLLECTION_ID }}
          LLM_PROVIDER: ${{ secrets.LLM_PROVIDER }}
          LLM_API_KEY: ${{ secrets.LLM_API_KEY }}
          LLM_MODEL: ${{ secrets.LLM_MODEL }}
        run: python -m src run

      - name: Commit & Push
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add docs/ data/articles/ data/output/ state/
          git diff --cached --quiet && echo "No changes to commit" || \
            (git commit -m "🤖 記事を要約・HTML を更新 $(date -u +'%Y-%m-%d')" && git push)

      - uses: actions/upload-pages-artifact@56afc609e74202658d3ffba0e8f6dda462b719fa  # v3
        with:
          path: docs/

  deploy:
    needs: summarize
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@d6db90164ac5ed86f2b6aed7e0febac5b3c0c03e  # v4
```

設定のポイントをいくつか補足する。

- **SHAピン留め**: すべてのActionsをコミットハッシュで固定し、サプライチェーン攻撃のリスクを軽減している
- **concurrency制御**: `cancel-in-progress: true` により、手動実行とcron実行が重複した場合は先行ジョブをキャンセルする
- **permissions の最小化**: `contents: write`（コミット＆プッシュ用）、`pages: write` + `id-token: write`（GitHub Pagesデプロイ用）のみに絞っている
- **変更なし時のスキップ**: `git diff --cached --quiet` で差分がなければコミットせず、不要な空コミットを防止
- **Secrets管理**: APIキーやトークンはすべてGitHub Secretsに格納し、ワークフロー内で環境変数として注入。ローカルでは `.env` で管理していた値をそのままSecretsに登録する形で、コードの変更なしにCI環境へ移行できた

登録しているSecretsは以下の5つ。

| Secret | 用途 |
|---|---|
| `RAINDROP_TOKEN` | Raindrop.io APIのアクセストークン |
| `RAINDROP_COLLECTION_ID` | 対象コレクションのID |
| `LLM_PROVIDER` | 使用するLLMプロバイダー（gemini） |
| `LLM_API_KEY` | LLMプロバイダーのAPIキー |
| `LLM_MODEL` | 使用するモデル名 |
- **2ジョブ構成**: summarize（要約＋コミット）→ deploy（GitHub Pagesデプロイ）を `needs` で直列化し、デプロイ順序を保証

## GitHubのRelease機能によるリリースノート作成

v1.0.0の完成後、GitHubのRelease機能を使ってリリースノートを作成した。

> **自分**: githubのRelease機能でリリースノートを作ってほしい
>
> **Claude**: 初リリースですね。全コミット履歴を元にリリースノートを作成します。

Claude Codeは `git log` で全コミット履歴を取得し、内容を分析した上で `gh release create` コマンドでリリースを作成してくれた。Features、Security、Documentation、Bug Fixesの4カテゴリに分類し、関連Issue番号（`#7`, `#34` など）も自動で紐づけてくれている。

![](https://storage.googleapis.com/zenn-user-upload/4367296e8403-20260328.png)

https://github.com/unsolublesugar/tsuyu-mi/releases/tag/v1.0.0

Issue作成からPR作成、リリースノート、READMEの整備まで、開発にまつわる一連の作業をすべてClaude Codeで完結できるのは実に快適だ。

## おわりに

「あとで読む（読まない）」を気合いの問題として抱えるのではなく、読み始めるまでの摩擦を減らす仕組みに変える。Tsuyu-miはそのアプローチのひとつの形だ。

開発にかかった時間は半日程度。思いついた構想をChatGPTで仕様に落とし込み、Claude Codeで実装からリリースまでを一気通貫で進められた。仕様駆動開発、Skill・Rulesの活用、Issue管理まで含めた開発ワークフロー全体をAIと回せるのは、個人開発との相性が非常に良いと感じた。

あくまで自分用のサービスではあるが、こうした「自分の課題を解決するツール」をAI駆動でサクッと作れるのは、とても良い時代になったと思う。

本プロジェクトの全容はGitHubにてMITライセンスで公開しているので、興味があればリポジトリを覗いてみてほしい。

https://github.com/unsolublesugar/tsuyu-mi

これを参考に、自分の「あとで読む」問題を解決するツールを作ってみるのも面白いと思う。

## 参考

https://note.com/unsoluble_sugar/n/n543659f80c1e

https://www.docswell.com/s/unsoluble_sugar/KWM7R1-2025-11-24-161918#p21