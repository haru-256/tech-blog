# AGENTS.md リポジトリ全体化 実装プラン

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** AGENTS.md をレビュー専用からリポジトリ全体を対象とした内容に刷新し、記事執筆・レビューの専門的な内容を skills/ に分離する

**Architecture:** AGENTS.md はリポジトリ概要・ワークフロー・スキル案内のみに絞る。詳細な執筆方法は `skills/write-article.md` へ、レビュー方法は `skills/review-article.md` へ移動する。`skills/review-criteria.md` は変更なし（review-article.md から参照される）。

**Tech Stack:** Markdown のみ

---

## ファイル構成マップ

| ファイル | 操作 | 役割 |
|---|---|---|
| `AGENTS.md` | 書き換え | リポジトリ全体の概要・ワークフロー・スキル案内 |
| `skills/write-article.md` | 新規作成 | 記事執筆スキル（テンプレート活用・スタイル） |
| `skills/review-article.md` | 新規作成 | 記事レビュースキル（現AGENTS.mdの内容を移動） |
| `skills/review-criteria.md` | 変更なし | 詳細なレビューチェックリスト |

---

### Task 1: `skills/review-article.md` を作成する

現在の `AGENTS.md` の内容（レビュー専用）を `skills/review-article.md` に移動する。

**Files:**
- Create: `skills/review-article.md`

- [ ] **Step 1: `skills/review-article.md` を作成する**

以下の内容で作成する（現 AGENTS.md から内容を抽出・整理）:

```markdown
# 記事レビュースキル

このスキルは、テックブログ記事のレビュー方法を定義します。
記事のレビューを依頼された際に参照してください。

## 1. レビューの優先度

各チェック項目には以下の優先度が設定されています:

- **MUST**: 必ず修正すべき重大な問題（技術的誤り、誤解を招く表現など）
- **SHOULD**: 記事の質を大きく向上させる推奨事項（構成の改善、説明の追加など）
- **MAY**: あれば望ましい改善提案（表現の洗練、図表の追加など）

## 2. レビューの参照先

詳細なチェック項目は以下を参照してください:

- **[review-criteria](review-criteria.md)**: レビューの観点、記事種別ガイドライン、引用とクレジット

## 3. レビュー対象外の項目

- **個人的なスタイルの選択**: 技術的に正しければ、筆者の好みを尊重
- **主観的な好み**: 「この技術が好き」「このフレームワークを使いたい」といった個人の嗜好
- **記事のテーマ選定**: 筆者が書きたいテーマを自由に選ぶ権利を尊重

## 4. フィードバックの方針

- **具体的に**: 「わかりにくい」ではなく、どこがどう分かりにくいのかを明示
- **建設的に**: 問題点と改善案をセットで提示
- **肯定的に**: 良かった点も積極的に評価し、執筆者のモチベーションを高める
- **簡潔に**: 1コメント1ポイントを原則とする

### 4.1. コメントの構造

レビューコメントは以下の構造で記載します:

\```markdown
## レビュー結果

### 良かった点 ✅

- [具体的な良かった箇所と理由]

### 改善提案 (MUST) 🔴

- **[セクション名]**: [問題点]
  - 改善案: [具体的な修正案]

### 改善提案 (SHOULD) 🟡

- **[セクション名]**: [問題点]
  - 改善案: [具体的な修正案]

### 改善提案 (MAY) 🔵

- **[セクション名]**: [問題点]
  - 改善案: [具体的な修正案]
\```

### 4.2. トーンとマナー

- 敬語を使用し、丁寧な表現を心がける
- 批判的な表現は避け、「〜してください」ではなく「〜すると良いでしょう」
- 筆者の努力や工夫を認める言葉を添える

## 5. レビュープロセス

1. **全体の把握**: まず記事全体を通読し、目的とターゲット読者を理解
2. **優先度の高い項目から**: MUST項目（技術的正確さ、コード品質など）を優先的にチェック
3. **セクションごとにレビュー**: 各セクションを順番にチェック
4. **総評の作成**: 記事全体の印象と主要な改善点をまとめる

### 5.1. レビュー完了後のチェックリスト

- [ ] MUST項目はすべて確認したか
- [ ] 良かった点を少なくとも3つ挙げたか
- [ ] すべての指摘に改善案を添えたか
- [ ] レビューコメントの口調は丁寧で建設的か
- [ ] 優先度（MUST/SHOULD/MAY）を適切に設定したか
- [ ] 記事の種類（プログラミング系/論文解説）に応じた適切なチェック項目を使用したか
```

- [ ] **Step 2: コミット**

```bash
git add skills/review-article.md
git commit -m "feat: add review-article skill extracted from AGENTS.md"
```

---

### Task 2: `skills/write-article.md` を作成する

記事の執筆方法を定義するスキルを新規作成する。テンプレートの活用方法・タイトル規約・執筆スタイルを含む。

**Files:**
- Create: `skills/write-article.md`
- Reference: `templates/` ディレクトリ（4種のテンプレート）

- [ ] **Step 1: `skills/write-article.md` を作成する**

```markdown
# 記事執筆スキル

このスキルは、テックブログ記事の執筆方法を定義します。
記事の作成・編集を行う際に参照してください。

## 1. 記事の種類とテンプレート

以下のテンプレートを用途に応じて使用してください:

| 種類 | テンプレート |
|---|---|
| プログラミング技術紹介 | `templates/IT技術紹介記事テンプレート.md` |
| 特定機能の紹介・解説 | `templates/IT特定機能紹介記事テンプレート.md` |
| トラブルシューティング | `templates/ITトラブルシューティング記事.md` |
| 論文解説 | `templates/論文解説記事テンプレート.md` |

## 2. ファイル管理

- **下書き**: `draft/` ディレクトリに `DRAFT - <記事タイトル>.md` 形式で保存
- **完成記事**: `articles/` ディレクトリに移動

## 3. タイトルの付け方

タイトルは記事の内容を正直に説明するものにする。以下のパターンは避ける:

- 問いかけ形式の煽り（「〜を知っていますか？」「〜できますか？」）
- 数字リスト系の過剰な強調（「絶対に使うべき10の方法」）
- 不安を煽る表現（「〜しないと損する」「〜は時代遅れ」）

**良い例:**
- `uvでPyTorchの環境構築を楽にする - OSやGPUの違いを吸収する統一的な方法`
- `connect-goインターセプタ実装パターンガイド - Unary/Streaming対応`

**悪い例:**
- `知らないと損する！PyTorchの環境構築の罠`
- `今すぐやめるべき5つのGo実装パターン`

## 4. 執筆スタイル

### 4.1. 対象読者

記事の冒頭で対象読者と前提知識を明示する。

### 4.2. コードサンプル

- 言語・バージョン情報を明記する
- 実行可能なコードを掲載する
- 重要な部分はコメントで補足する

### 4.3. 引用・参考資料

- 公式ドキュメントへのリンクを積極的に貼る
- 論文解説記事では DOI または arXiv ID を必ず記載する
- 自分の解釈と引用元の記述を明確に区別する

## 5. 下書きから完成までの流れ

1. `draft/` に下書き作成
2. 自己レビュー（review-article スキル参照）
3. 内容が固まったら `articles/` に移動
```

- [ ] **Step 2: コミット**

```bash
git add skills/write-article.md
git commit -m "feat: add write-article skill for article authoring guidance"
```

---

### Task 3: `AGENTS.md` をリポジトリ全体向けに書き換える

レビュー専用の内容を削除し、リポジトリ概要・ワークフロー・スキルの案内に絞る。

**Files:**
- Modify: `AGENTS.md`

- [ ] **Step 1: `AGENTS.md` を書き換える**

以下の内容で全面的に書き換える:

```markdown
# テックブログ エージェント指示

## 1. このリポジトリについて

Zennに投稿するテックブログ記事とサンプルコードを管理するリポジトリです。
主なワークフローは **記事を書く** と **記事をレビューする** の2つです。

## 2. ディレクトリ構成

```
.
├── articles/          # 完成した記事（Zennに投稿済み、または投稿予定）
├── books/             # Zennの本形式コンテンツ
├── draft/             # 執筆中の下書き（DRAFT - <タイトル>.md 形式）
├── images/            # 記事で使用する画像
├── templates/         # 記事テンプレート（種類別）
└── skills/            # タスク別の専門スキル
    ├── write-article.md    # 記事執筆スキル
    ├── review-article.md   # 記事レビュースキル
    └── review-criteria.md  # レビュー詳細基準
\```

## 3. スキルの使い方

タスクに応じて以下のスキルを参照してください:

| タスク | 参照するスキル |
|---|---|
| 記事を書く・下書きを作る | [write-article](skills/write-article.md) |
| 記事をレビューする | [review-article](skills/review-article.md) |

## 4. 一般的な規約

- 下書きは `draft/` に保存し、完成後に `articles/` へ移動する
- 記事ファイル名は記事タイトルと同じにする
- 画像は `images/` に置き、記事から相対パスで参照する
```

- [ ] **Step 2: コミット**

```bash
git add AGENTS.md
git commit -m "refactor: rewrite AGENTS.md as repo-wide agent guide, move specifics to skills"
```

---

## Self-Review

**Spec coverage:**
- [x] AGENTS.md をリポジトリ全体対象にする → Task 3 で実装
- [x] 記事を書く専門内容を skills に移す → Task 2 (write-article.md)
- [x] 記事をレビューする専門内容を skills に移す → Task 1 (review-article.md)
- [x] 既存の review-criteria.md は変更なし → review-article.md から参照

**Placeholder scan:** なし

**Type consistency:** Markdown ファイルのみのため型の一貫性は不要
