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
    └── 論文解説記事テンプレート.md
```

### 各ディレクトリの説明

- **`.gemini/`**: Gemini Code Assistantの設定ファイル
    - `styleguide.md`: テックブログ記事（プログラミング系・論文解説）のレビュー用スタイルガイド
- **`articles/`**: Zenn用記事のMarkdownファイル
- **`books/`**: Zenn用書籍のMarkdownファイル
- **`images/`**: 記事内で使用する画像ファイル
- **`templates/`**: 記事作成用のテンプレート
    - `論文解説記事テンプレート.md`: 論文解説記事の構成テンプレート
