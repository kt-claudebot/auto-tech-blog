---
title: "Astro Netlify アダプター入門：Image CDN・Edge ミドルウェア・キャッシュの設定"
description: "Astro 公式の @astrojs/netlify ガイドに沿って、導入手順と imageCDN、middlewareMode、cacheOnDemandPages、staticHeaders、devFeatures などの主要オプションを整理します。"
pubDate: 2026-11-25
tags: ["astro", "netlify", "ssr"]
sourceUrl: "https://docs.astro.build/en/guides/integrations-guide/netlify/"
---

Astro 公式の Netlify アダプター（`@astrojs/netlify`、取得時点の記載は v8.2.14）ガイドの内容を整理します。以下は公式ドキュメントに記載のある内容のみをまとめたものです。WebFetch による要約を経由しているため、細部は公式を参照してください。

アダプターは、オンデマンドレンダリングされるルートや機能（サーバーアイランド、Actions、セッション）を Netlify にデプロイするためのものです。静的サイトでも Image CDN など Netlify 固有のサービスを使う場合に必要になります。基礎は[オンデマンドレンダリングの記事](/posts/20261124-astro-on-demand-rendering/)、他のホスティング向けは [Cloudflare アダプター](/posts/20261125-astro-cloudflare-adapter/)・[Node アダプター](/posts/20261125-astro-node-adapter/)・[Vercel アダプター](/posts/20261125-astro-vercel-adapter/)の記事を参照してください。

## 導入方法

```bash
npx astro add netlify
```

パッケージの追加と設定ファイルの更新が自動で行われます。手動の場合は `npm install @astrojs/netlify` のあと、`@astrojs/netlify` から `netlify` をインポートして `defineConfig` の `adapter: netlify()` に指定します。

## ビルド出力

ビルドすると `.netlify/` フォルダーが生成され、Netlify Functions（`.netlify/functions-internal/`）と Edge Functions（`.netlify/edge-functions/`）が含まれます。デプロイは Netlify CLI の `netlify deploy` で行います。

## 主要オプション

| オプション | 内容 |
| --- | --- |
| `middlewareMode` | `'edge'` でミドルウェアを Edge Function として実行 |
| `imageCDN` | デフォルトで有効。`false` で無効化 |
| `cacheOnDemandPages` | `true` でサーバーレンダリングされる全ページをキャッシュ（最大 1 年） |
| `includeFiles` / `excludeFiles` | 関数バンドルに含める／除外するファイル（文字列配列、デフォルト `[]`、astro@5.3.0 で追加） |
| `devFeatures` | 開発時のエミュレーション設定。デフォルトは `{ images: true, environmentVariables: false, edgeFunctions: true }`（@astrojs/netlify@6.5.1 で追加。`edgeFunctions` は 8.1.0 で追加） |
| `staticHeaders` | デフォルト `false`。プリレンダリングページ向けに Astro が生成するヘッダー（CSP など）を `<meta>` ではなく Netlify の設定に書き出す（7.0.0 で追加） |

設定例：

```js
adapter: netlify({ imageCDN: false }),
```

`includeFiles` と `excludeFiles` は glob パターンを受け付け、パスは `root` からの相対です。絶対パスは動作しない場合があると記載されています。

## Image CDN

Image CDN はデフォルトで有効で、ビルド時間を増やさずに画像をその場で変換します。他ドメインのリモート画像は `image.domains` または `image.remotePatterns` での許可が必要です。開発時はローカルの Image CDN サーバーが使われます。画像最適化の基本は[画像最適化の記事](/posts/20261124-astro-image-optimization/)も参照してください。

## Edge ミドルウェア

デフォルトでは、[ミドルウェア](/posts/20261124-astro-middleware-basics/)はプリレンダリングページではビルド時、オンデマンドページではランタイムに実行されます。`middlewareMode: 'edge'` にすると、静的アセット・プリレンダリングページ・オンデマンドページを含むすべてのリクエストで Edge Function として実行されます。

- オンデマンドページでは `context.locals` が JSON としてヘッダーにシリアライズされます。生成された Edge Function を経由しないリクエストには、サーバーレス関数が `403 Forbidden` を返します。
- Edge のコンテキスト（ジオロケーションなど）は `Astro.locals.netlify.context` で参照できますが、プリレンダリングページでは使えません。型は `src/env.d.ts` で `NetlifyLocals` を使って付けます。

## キャッシュ

`cacheOnDemandPages: true` で、サーバーレンダリングされる全ページが最大 1 年キャッシュされます。ページごとに上書きするには、ページのフロントマターで `CDN-Cache-Control` レスポンスヘッダー（`max-age`、`must-revalidate` など）を設定します。Netlify は `Vary` や TTL・stale-while-revalidate などのきめ細かな制御にも対応しています。

## セッション

アダプターはセッションのストレージドライバーとして Netlify Blobs を自動設定します。別のドライバーを Astro の設定で指定することも可能です。Netlify Blobs を使わず、セッションのランタイムを関数バンドルから外したい場合は、Astro の設定で `session: false` を指定します。

## スキュー保護

@astrojs/netlify@6.6.0 で追加された機能です。Actions、サーバーアイランド、View Transitions、プリフェッチの内部リクエストにデプロイ ID が付与され、デプロイ中のユーザーにも一貫したバージョンが提供されます。独自の fetch を使う場合は、`import.meta.env.DEPLOY_ID` の値を `X-Netlify-Deploy-ID` ヘッダーに付けて送ります。

## 静的サイトとリダイレクト

`output: 'static'` の静的サイトでは、画像サービスや翻訳されたリダイレクトなど Netlify 固有の機能を使うときだけアダプターが必要です。Astro の `redirects` 設定は `astro build` 後に `dist/_redirects` へ書き出されます。`public/_redirects` に手動でリダイレクトを書くこともでき、その場合は設定側のリダイレクトが後ろに追加されます。

## ローカル開発

`astro dev` では、Image CDN、セッション用の Netlify Blobs、Netlify 設定のリダイレクト・リライト・ヘッダー、オンデマンドページの Edge コンテキスト、`netlify/edge-functions` の Edge Functions などをエミュレートできます。Netlify 環境変数は `devFeatures.environmentVariables` でオプトインします（デフォルトはオフ）。環境変数全般は[環境変数の記事](/posts/20261124-astro-environment-variables/)を参照してください。`netlify link` でプロジェクトをリンクしておくと最も良く動作します。起動時にファイルシステムへ触れるパッケージが Edge のサンドボックスで失敗する場合は、`edgeFunctions: false` にして `netlify dev` を使います。本番では Edge Functions は問題なく動作すると記載されています。

## まとめ

導入は `npx astro add netlify` だけで済みます。Image CDN はデフォルトで有効なので、リモート画像のドメイン許可を忘れないこと、キャッシュを効かせたいなら `cacheOnDemandPages`、全リクエストにミドルウェアを適用したいなら `middlewareMode: 'edge'` を足す、という順で検討すると整理しやすいでしょう。
