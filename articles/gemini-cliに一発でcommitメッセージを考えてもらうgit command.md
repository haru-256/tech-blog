---

title: "gemini-cliに一発でcommitメッセージを考えてもらうgit command" # zenn: 記事のタイトル
emoji: "😸" # zenn: アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # zenn: tech: 技術記事 / idea: アイデア記事
topics: [] # zenn: タグ。["markdown", "rust", "aws"]のように指定する
published: false # zenn: 公開設定（falseにすると下書き）
tags: [] # paper: tags
---

3行まとめ

1. gemini-cliを使って、ステージングされた変更から自動でcommitメッセージを生成する方法を紹介
2. gitのフックスクリプトとして利用することで、コミット時に自動でメッセージを生成可能。そのままcommitメッセージを編集できるので、手直しも簡単
3. ただし、gemini-cliの起動には時間がかかるため、頻繁なコミットには向かない点に注意

## 1. はじめに

この記事で紹介する機能（[機能名]）が、[技術名]においてどのような位置づけ・役割を持つ機能なのかを簡潔に説明する。

この記事を読むことで読者が何を得られるのか（記事のゴール）。

対象読者（例：「[技術名]を使っていて、[機能名]について詳しく知りたい人」「[特定の課題]を[機能名]で解決したい人」など）。

### 前提条件

- gemini-cliがインストールされ、動作する環境であること
- gitがインストールされ、基本的な操作ができること

## 2. [機能名]とは？（なぜこの機能が必要か）

[機能名]の概要と、それが解決しようとしている課題（この機能がないと何が大変か）。

この機能を使うことの主なメリットや特徴を挙げる。

## 3. git commandでcommitメッセージを自動生成する方法

- 実際のスクリプトはこちら

```bash
#!/usr/bin/env bash

set -e
set -o pipefail

# 1. ステージング確認
if git diff --cached --quiet; then
    echo "🚫 ステージングされた変更がありません。"
    exit 1
fi

echo "🤖 Geminiがコミットメッセージを生成中..."

# 2. プロンプト設定
PROMPT="以下のgit diffから、Conventional Commitsに従ったコミットメッセージを作成してください。タイトルと箇条書きの本文を含め、Markdownのコードブロック(backticks)は含めず、プレーンテキストのみで出力してください。"

# 3. 実行
git diff --cached | gemini "$PROMPT" | git commit -e -F -
```

- 上記スクリプトを適当な位置に配置し、実行権限を付与

```bash
mv path/to/your/script.sh ~/.local/bin/git-commit-with-gemini.sh # パスが通っている場所に移動
chmod +x ~/.local/bin/git-commit-with-gemini.sh # 実行権限付与
```

- git aliasとして登録

```bash
git config --global alias.commit-with-gemini '!~/.local/bin/git-commit-with-gemini.sh'
```

## 4. 基本的な使い方

[機能名]を有効にするための最小限のセットアップや、最も基本的な使い方を示すコード例。

（コード例）

（実行結果や動作の解説）

## 5. 注意点

実際に使う上で考慮すべき点（パフォーマンス、実行順序、エラー処理など）。

陥りがちな罠や、推奨される設計パターンなど。

## 6. まとめ

記事全体の内容を簡潔に振り返り、[機能名]の有用性を再確認する。

さらに学ぶための参考リンク（公式ドキュメントの該当ページ、関連ライブラリなど）を紹介する。
