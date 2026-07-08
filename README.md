# my-zenn

[Zenn](https://zenn.dev/) の記事を GitHub 連携で管理するリポジトリです。

## セットアップ

```bash
# mise で Node.js をインストール
mise install

# 依存関係をインストール
npm install
```

## 記事作成〜公開のフロー

```bash
# 1. 作業ブランチを作成
git checkout -b article/my-article-slug

# 2. 新しい記事を作成（または templates/article-template.md をコピー）
npm run new

# 3. ローカルでプレビュー
npm run preview
# → http://localhost:8000 で確認

# 4. 執筆・レビュー（published: false のまま作業）

# 5. 公開する場合
#    - frontmatter の published を true に変更
#    - main ブランチにマージして push
#    → Zenn が自動で記事を公開
```

## ディレクトリ構成

```
articles/   # 記事（Markdown）
books/      # 本（Markdown）
templates/  # 記事テンプレート
```

## AI 執筆支援

記事作成を AI に手伝ってもらう際のルールは [CLAUDE.md](./CLAUDE.md) を参照してください。
