---
title: "Astro Vercel アダプター入門：ISR・画像最適化・Edge ミドルウェアの設定"
description: "Astro 公式の @astrojs/vercel ガイドに沿って、導入手順と webAnalytics、imageService、isr、maxDuration、skewProtection、middlewareMode などの主要オプションを整理します。"
pubDate: 2026-11-25
tags: ["astro", "vercel", "ssr"]
sourceUrl: "https://docs.astro.build/en/guides/integrations-guide/vercel/"
---

Astro 公式の Vercel アダプター（`@astrojs/vercel`）ガイドの内容を整理します。以下は公式ドキュメントに記載のある内容のみをまとめたものです。WebFetch による要約を経由しているため、細部は公式を参照してください。

静的サイトだけなら、アダプターはオンデマンドレンダリングや Vercel の追加サービス（Web Analytics、Image Optimization）を使う場合にのみ必要です。基礎は[オンデマンドレンダリングの記事](/posts/20261124-astro-on-demand-rendering/)、他のホスティング向けは[Cloudflare アダプター](/posts/20261125-astro-cloudflare-adapter/)・[Node アダプター](/posts/20261125-astro-node-adapter/)の記事を参照してください。

## 導入方法

```bash
npx astro add vercel
```

パッケージのインストールと設定ファイルの更新が行われます。手動の場合は `npm install @astrojs/vercel` のあと、設定にアダプターを追加します。

```js
import { defineConfig } from 'astro/config';
import vercel from '@astrojs/vercel';
export default defineConfig({
  // ...
  adapter: vercel(),
});
```

## 主要オプション

| オプション | 内容 |
| --- | --- |
| `webAnalytics` | `{ enabled: true }` で計測スクリプトを挿入（3.8.0 で追加）。`@vercel/analytics` 1.4.0 以降は Vercel の Analytics コンポーネントを使うよう記載あり |
| `imageService` | `true` で本番に Vercel Image Optimization を使用（3.3.0 で追加） |
| `devImageService` | 開発時の画像サービス。デフォルトは `'sharp'`（3.8.0 で追加） |
| `imagesConfig` | Vercel Image Optimization API へのオプション。`domains` と `remotePatterns` は Astro の `image` 設定から補完（3.3.0 で追加） |
| `isr` | デフォルト `false`。オンデマンドページを ISR 関数としてデプロイし、初回リクエスト後にキャッシュ（7.2.0 で追加） |
| `includeFiles` / `excludeFiles` | 関数バンドルへ強制的に含める／除外するファイル（文字列配列） |
| `maxDuration` | 関数のタイムアウト秒数（例: `60`）。上限は Vercel のプランに依存 |
| `skewProtection` | `true` で Vercel の Skew protection を有効化。Pro または Enterprise が必要（7.6.0 で追加） |
| `staticHeaders` | デフォルト `false`。CSP などの静的ヘッダーを `vercel.json` に書き出す（10.0.0 で追加） |
| `middlewareMode` | `'edge'` でミドルウェアを Edge Function として実行 |

画像まわりの考え方は[画像最適化の記事](/posts/20261124-astro-image-optimization/)も参照してください。

## ISR の詳細

`isr` にはオブジェクトを渡せます。

- `expiration`: キャッシュの有効秒数（例: `60 * 60 * 24`）。Cache-Control ヘッダーも尊重されます。
- `bypassToken`: `x-prerender-revalidate` ヘッダーによるオンデマンドの無効化に使います。
- `exclude`: 常に最新を描画するパス。正規表現は 8.1.0 から利用可能です。
- ドラフトモードは `__prerender_bypass` クッキーで使えます。

## Edge ミドルウェア

`middlewareMode: 'edge'` を指定すると、[Astro のミドルウェア](/posts/20261124-astro-middleware-basics/)が、オンデマンド／ISR ルートの前段で Vercel Edge Function として動き、キャッシュヒット時にも実行されます。Edge の locals は `EdgeLocals` で型付けします。エッジ関数を経由しないリクエストに対しては、サーバーレス関数が 403 を返すと記載されています。

## セッション

`session` 設定でストレージドライバー（例: `ioredis` 経由の Redis）を指定します。ローカルで認証情報を読み込むには `vercel env pull .env.local` を使います。環境変数全般は[環境変数の記事](/posts/20261124-astro-environment-variables/)を参照してください。

## まとめ

導入は `npx astro add vercel` だけで済みます。ISR でキャッシュを効かせたい、画像最適化を Vercel に任せたい、といった要件に応じて `isr` や `imageService` を足していく形です。Node のバージョンは Vercel プロジェクト設定の「Node.js Version」に一覧があります。
