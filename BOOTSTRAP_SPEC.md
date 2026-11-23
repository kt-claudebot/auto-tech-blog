# 全自動技術ブログ基盤 構築仕様書（ルールブック）

あなたはこのディレクトリ（`/home/claude-bot/workspace/auto-tech-blog`）に、完全自動で記事生成・公開を行うための技術ブログ基盤をゼロから構築します。

## 1. 技術スタック・要件
- **フレームワーク:** Astro（最新の安定版、公式ブログテンプレートまたは最小構成＋Content Collections）
- **言語:** TypeScript（strict）
- **デプロイ先想定:** Cloudflare Pages（静的サイト出力 `output: 'static'`）
- **必須機能:**
  1. 記事一覧ページ（日付降順）、記事詳細ページ、タグ別一覧ページ
  2. SEO基本対応（`sitemap`、`rss.xml`、OGPメタタグ、description）
  3. 収益化用スロット：記事上部・記事下部に広告やアフィリエイトリンクを差し込める共通コンポーネント（`src/components/AdSlot.astro`）を用意（初期状態はコメントアウトされたプレースホルダー）
  4. **厳格なスキーマ検証（Zod）:** `src/content/config.ts` でフロントマター（`title`, `description`, `pubDate`, `tags`, `sourceUrl`）を必須チェックし、不正な記事は `npm run build` で確実にエラーになるようにする。

## 2. 運用ルールブック（`CLAUDE.md`）の自動生成
リポジトリ直下に、今後の自律記事生成・自動レビュー時にClaude Codeが読み込む `CLAUDE.md` を作成すること。以下を必ず含める：
- 記事の保存先（`src/content/posts/YYYYMMDD-<slug>.md`）とフロントマターの正確なフォーマット例
- 一次ソース（`sourceUrl`）に記載のない仕様・コマンドの捏造禁止
- 既存記事との重複チェック手順
- コミット前に必ず `npm run build` を実行し、エラー0を確認すること
- `main` ブランチへの直接プッシュ禁止（必ずブランチを切って `gh pr create` すること）

## 3. 初期コンテンツと検証
- 動作確認用のサンプル記事を1本（`src/content/posts/20261124-hello-world.md`）作成する。
- `npm run build` を実行し、`dist/` がエラーなく生成されることを確認する。

## 4. Git / GitHub 連携
- `.gitignore`（`node_modules/`, `dist/`, `.astro/`, `.env` 等）を適切に設定する。
- `git init` し、初期コミットを作成する。
- `gh repo create auto-tech-blog --public --source=. --remote=origin --push` を実行してGitHubへプッシュする（※すでにリポジトリが存在する場合はエラーを報告して止める）。

## 5. 最終報告
完了後、以下の3点のみを簡潔に出力すること：
1. 作成したGitHubリポジトリのURL
2. `npm run build` の結果
3. 生成した `CLAUDE.md` と主要ファイルのパス一覧
