# Tech Blog

Tech Blogのための記事やサンプルコードなどのコンテンツを管理するリポジトリ。

## ディレクトリ構成

```sh
$ tree -d -L 2 -I '.git|.obsidian|.vscode|node_modules'
.
├── .gemini           # Gemini Code Assistant設定
│   └── styleguide.md # 記事レビュー用スタイルガイド
├── articles          # Zenn記事ディレクトリ
├── books             # Zenn本ディレクトリ
├── images            # 記事用画像ディレクトリ
└── templates         # 記事テンプレート
    ├── ITトラブルシューティング記事.md
    ├── IT特定機能紹介記事テンプレート.md
    ├── IT技術紹介記事テンプレート.md
    └── 論文解説記事テンプレート.md
```

## 各記事の紹介

### 1. 論文解説 IntentRec - Predicting User Session Intent with Hierarchical Multi-Task Learning

**概要:**
Netflixが発表した推薦システムに関する手法IntentRecの解説記事です。ユーザーの「今、何をしたいか」という意図（新しい作品の発見、続きの視聴など）を予測し、その結果を次の推薦アイテム予測に活用する階層的マルチタスク学習アーキテクチャを提案。Netflixの実データでSOTA（最高性能）を達成した画期的な研究です。

**キーワード:** `Netflix` `推薦システム` `意図予測` `階層的マルチタスク学習` `機械学習`

**記事リンク:** [論文解説 IntentRec - Predicting User Session Intent with Hierarchical Multi-Task Learning](./articles/論文解説%20IntentRec%20-%20Predicting%20User%20Session%20Intent%20with%20Hierarchical%20Multi-Task%20Learning.md)

---

### 2. connect-goインターセプタ実装パターンガイド - Unary/Streaming対応

**概要:**
connect-goのインターセプタ実装パターンとライフサイクルを徹底解説した記事です。Unary RPCとStreaming RPCでは実装方法が根本的に異なり、Unaryは処理関数(`next`)をラップするのに対し、Streamingは接続オブジェクト(`conn`)をラップしてSend/Receiveメソッドをフックします。この記事では、ロギングを題材に、認証・メトリクス・レート制限など様々な横断的関心事に応用できる実装パターンを、クライアント側/サーバー側の違いを含めて詳しく解説します。

**キーワード:** `Go` `gRPC` `connect-go` `Interceptor` `AOP` `ロギング` `認証` `アーキテクチャ`

**記事リンク:** [connect-goインターセプタ実装パターンガイド - Unary/Streaming対応](./articles/connect-goインターセプタ実装パターンガイド.md)

---

### 3. DDDにおけるuber-fxによるDI

**概要:**
*(執筆中)*

**記事リンク:** [DDDにおけるuber-fxによるDI](./articles/DDDにおけるuber-fxによるDI.md)

---

### 4. uvでPyTorchの環境構築を楽にする - OSやGPUの違いを吸収する統一的な方法

**概要:**
PyTorchの環境構築を、OS（macOS/Linux）やGPU有無の違いを吸収して「make install」一発で自動化する方法を解説。Rust製パッケージ管理ツール`uv`のoptional-dependenciesとMakefileを組み合わせ、どんな環境でも最適なPyTorch（CPU/GPU版）が自動インストールされる仕組みを紹介します。チーム開発での環境差異によるトラブルや手順の煩雑さを解消し、再現性の高いセットアップを実現します。

**キーワード:** `Python` `PyTorch` `uv` `環境構築` `Makefile` `クロスプラットフォーム`

**記事リンク:** [uvでPyTorchの環境構築を楽にする - OSやGPUの違いを吸収する統一的な方法](./articles/uvでPyTorchの環境構築を楽にする%20-%20OSやGPUの違いを吸収する統一的な方法.md)

---

## テンプレートの解説

| ファイル | 用途 |
| --- | --- |
| `templates/IT技術紹介記事テンプレート.md` | 新しい技術やツールを紹介する記事向け。フロントマッター済みで、課題→解決→まとめの流れを簡単に構築できる。 |
| `templates/IT特定機能紹介記事テンプレート.md` | 既存プロダクトの特定機能（例: gRPC Interceptorなど）にフォーカスした記事向け。基本の使い方からユースケース、応用例まで掘り下げられる。 |
| `templates/ITトラブルシューティング記事.md` | 特定のエラーや不具合に直面した際の原因調査～解決策を整理するためのテンプレート。再現環境・ログ・原因・解決手順を一貫して記録できる。 |
| `templates/論文解説記事テンプレート.md` | 論文を読む際の要約・考察を効率化するためのテンプレート。論文情報、課題設定、手法、実験結果、所感を漏れなく整理できる。 |
