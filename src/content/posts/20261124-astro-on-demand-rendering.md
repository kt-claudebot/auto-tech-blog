---
title: "Astro オンデマンドレンダリング入門：アダプター・prerender・リクエストデータの扱い"
description: "Astro 公式ガイドに沿って、オンデマンドレンダリング（SSR）の有効化方法（アダプター、prerender = false、output: 'server'）と、Astro.request・Astro.cookies・Astro.response の使い方を整理します。"
pubDate: 2026-11-24
tags: ["astro", "ssr", "typescript"]
sourceUrl: "https://docs.astro.build/en/guides/on-demand-rendering/"
---

Astro 公式の On-demand rendering ガイドの内容を整理します。以下は公式ドキュメントに記載のある内容のみをまとめたものです。WebFetch による要約を経由しているため、細部や全コード例は公式を参照してください。

## オンデマンドレンダリングとは

ページやルートを、ビルド時ではなくリクエスト時にサーバーでレンダリングする方式です（サーバーサイドレンダリング、SSR とも呼ばれます）。アカウント情報や最新データなど、訪問者ごとに異なるレスポンスを、サイト全体を再ビルドせずに返せます。

## 有効化の方法

### 1. アダプターを追加する

アダプターは、特定のランタイム向けにスクリプトを出力するためのものです。公式アダプターには Cloudflare、Netlify、Node、Vercel があります。たとえば Netlify の場合は次のコマンドでインストールと設定更新ができます。

```sh
npx astro add netlify
```

パッケージを手動でインストールすることもできます。

### 2. 特定ページだけプリレンダリングをやめる

デフォルトではサイト全体がプリレンダリング（静的生成）されます。ページやエンドポイントに `export const prerender = false` を書くと、そのページだけオンデマンドで描画されます。

```astro
---
export const prerender = false;
---
<html>
  <!-- Rendered on demand -->
</html>
```

### 3. `output: 'server'` モード

`output: 'server'` を設定すると、すべてのページがデフォルトでオンデマンドになります。静的のままにしたいページには `export const prerender = true` を追加します。公式は、まずデフォルトの `'static'` モードで始めることを勧めており、`'server'` モードはデフォルトを変えるだけで追加機能はない、と説明しています。

## リクエストデータの扱い

- **`Astro.request`**: 標準の Request オブジェクトです。`headers` や `method` を読めます。例: `Astro.request.headers.get('cookie')`。
- **`Astro.cookies`**: Cookie の確認・取得・設定・削除ができます。公式の例では `has`・`get`・`set` でページビューのカウンターを実装しています。
- **`Astro.response`**: 標準の ResponseInit オブジェクトで、`status`・`statusText`・`headers` を設定できます。例として、商品が存在しない場合に 404 を設定する使い方が挙げられています。
- ページから `Response` を直接返すことや `Astro.redirect` も使えます。

## 注意点

- レスポンスヘッダーを変更する機能は、ページ単位でのみ動作し、コンポーネント（レイアウト含む）の中では動作しません。ヘッダーはコンポーネントのコードが実行される前に送信されるためです。
- サーバーエンドポイント（API ルート）は `src/pages/` 内でエクスポートした関数で、コンテキストを受け取り Response を返します。公式の例では JSON を返す非同期の `GET` 関数をエクスポートしています。
- HTML ストリーミングにより、各コンポーネントはレンダリングされ次第ブラウザへ送られます。

## 関連記事

オンデマンドレンダリングと組み合わせて使う機能として、[Astro Actions 入門](/posts/20261124-astro-actions-basics/)（フォームアクションにはオンデマンドレンダリングが必要）、[Astro ミドルウェア入門](/posts/20261124-astro-middleware-basics/)、[Astro の環境変数入門](/posts/20261124-astro-environment-variables/)も参照してください。
