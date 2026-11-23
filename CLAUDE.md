# 自動技術ブログ 運用ルールブック

Astro 6 + TypeScript(strict) + Content Collections の静的ブログ（Cloudflare Pages 想定）。
記事の自律生成・自動レビュー時は必ず本ファイルに従うこと。

> 注: Astro 7 のネイティブコンパイラは AVX2 非対応 CPU でクラッシュするため Astro 6 系に固定している。環境が許す場合のみ更新を検討する。

## 記事の保存先とフォーマット
- 保存先: `src/content/posts/YYYYMMDD-<slug>.md`（`YYYYMMDD` は pubDate と一致、`<slug>` は英小文字・数字・ハイフン）
- フロントマターは全項目必須（スキーマ: `src/content/config.ts`、未知のキーもエラー）:

```markdown
---
title: "記事タイトル"
description: "検索結果に表示される1〜2文の要約"
pubDate: 2026-11-24
tags: ["astro", "typescript"]
sourceUrl: "https://example.com/primary-source"
---

本文（Markdown）
```

- `tags` は1個以上、`sourceUrl` は有効な URL（一次ソース）。

## 内容のルール（捏造禁止）
- 仕様・コマンド・API・数値・バージョンは `sourceUrl`（一次ソース）に記載のあるものだけを書く。
- 一次ソースに書かれていないことを推測や記憶で補わない。確認できない内容は書かないか、推測と明記する。
- 実際に取得して読んだページのみを `sourceUrl` にする。コードやコマンドは原文から正確に転記する。

## 既存記事との重複チェック手順
1. `ls src/content/posts/` で既存ファイル名を確認する。
2. `grep -ril "<キーワード>" src/content/posts/` でトピックを検索する。
3. `grep -rl "<sourceUrl>" src/content/posts/` で同じ一次ソースの記事がないか確認する。
4. 重複する場合は新規作成せず、既存記事の更新または別トピックへ変更する。

## コミット・公開フロー
1. `main` へ直接 push しない。必ずブランチ（例: `post/YYYYMMDD-<slug>`）を切る。
2. コミット前に必ず `npm run build`（`astro check` + `astro build`）を実行し、エラー0を確認する。失敗したらコミットしない。
3. コミット → `git push -u origin <branch>` → `gh pr create` で PR を作成する。
4. マージは人間または自動レビューの承認後に行う。

## 主要ファイル
- `src/content/config.ts` … Zod スキーマ（`src/content.config.ts` は Astro 用の再エクスポート。削除しない）
- `src/components/AdSlot.astro` … 広告/アフィリエイト枠（記事上部・下部、初期はコメントアウト）
- `src/pages/` … 一覧・記事詳細・タグ別・`rss.xml`
