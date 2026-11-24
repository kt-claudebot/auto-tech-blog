---
title: "Astro Node アダプター入門：standalone / middleware モードと起動方法"
description: "Astro 公式の @astrojs/node ガイドに沿って、導入手順、standalone と middleware モードの違い、HOST/PORT の指定、bodySizeLimit などの主要オプション、環境変数の扱いを整理します。"
pubDate: 2026-11-25
tags: ["astro", "node", "ssr"]
sourceUrl: "https://docs.astro.build/en/guides/integrations-guide/node/"
---

Astro 公式の Node アダプター（`@astrojs/node`）ガイドの内容を整理します。以下は公式ドキュメントに記載のある内容のみをまとめたものです。WebFetch による要約を経由しているため、細部は公式を参照してください。

このアダプターは、オンデマンドレンダリングのルート、サーバーアイランド、アクション、セッションを Node 環境で動かすためのものです。静的サイトだけならアダプターは不要です。基礎は[オンデマンドレンダリングの記事](/posts/20261124-astro-on-demand-rendering/)、Cloudflare を使う場合は[Cloudflare アダプターの記事](/posts/20261125-astro-cloudflare-adapter/)を参照してください。

## 導入方法

公式は `astro add node` を案内しています（npm / pnpm / Yarn いずれも可）。パッケージのインストールと設定ファイルの更新が行われます。

```bash
npx astro add node
```

手動の場合は `@astrojs/node` をインストールし、`astro.config.mjs` でアダプターを `node({ mode: 'standalone' })` の形で指定します。

```js
import { defineConfig } from 'astro/config';
import node from '@astrojs/node';

export default defineConfig({
  adapter: node({ mode: 'standalone' }),
});
```

（import 文を含む全体は、公式の記述にある `node({ mode: 'standalone' })` を基にした最小構成の例です。）

## 主要オプション

| オプション | デフォルト | 内容 |
| --- | --- | --- |
| `mode` | なし | `'middleware'` または `'standalone'` |
| `staticHeaders` | `false` | CSP などのヘッダーをプリレンダリングページに `Response` 経由で適用（10.0.0 で追加） |
| `experimentalDisableStreaming` | `false` | オンデマンドページの HTML ストリーミングを無効化（9.3.0 で追加。通常は非推奨と記載） |
| `bodySizeLimit` | 1 GB（1073741824 バイト） | リクエストボディの上限。`Infinity` か `0` で無制限（10.0.0 で追加） |

なお、ホストとポートはアダプターのオプションではなく、実行時の環境変数で指定します。

## mode の違い

- **standalone**: エントリーモジュールを実行すると自動でサーバーが起動します。静的ファイルも配信します。
- **middleware**: 他の Node サーバー（公式には Express や Fastify の例あり）の中で使うモードです。`./dist/server/entry.mjs` が `handler` 関数をエクスポートします。このモードでは静的ファイルは配信されないため、`./dist/client/` をフレームワーク側で配信する必要があります。`handler` の第4引数に locals オブジェクトを渡すと、`Astro.locals` や[ミドルウェア](/posts/20261124-astro-middleware-basics/)から参照できます。

## standalone サーバーの起動

```bash
node ./dist/server/entry.mjs
```

ホストとポートを指定する例です。

```bash
HOST=0.0.0.0 PORT=4321 node ./dist/server/entry.mjs
```

HTTPS を使うには `SERVER_KEY_PATH` と `SERVER_CERT_PATH` に鍵と証明書のパスを設定します。未設定なら HTTP で動作し、HTTPS プロキシの背後に置く構成に向くと公式に記載があります。

## 静的アセットとキャッシュ

standalone では `dist/client/` のファイルがサーバーから配信されます（CDN から配信することも可能）。`dist/client/_astro/` 内のハッシュ付きファイルには `Cache-Control: public, max-age=31536000, immutable` が付きます。

## セッションと環境変数

- セッションはデフォルトでローカルファイルシステムを使います。別のドライバーも設定可能です。無効化するには Astro 設定で `session: false` とします。
- Astro もアダプターも、実行時に環境変数を自動では読み込みません。セルフホストでは、インラインで指定する、dotenvx を使う、Dockerfile で設定する、といった方法が挙げられています。dotenvx の例は次のとおりです。

```bash
npx @dotenvx/dotenvx run -- node ./dist/server/entry.mjs
```

Astro 側の環境変数の扱いは[環境変数の記事](/posts/20261124-astro-environment-variables/)も参照してください。

## まとめ

Node で単体運用するなら `standalone`、既存の Express などに組み込むなら `middleware` を選びます。standalone は `node ./dist/server/entry.mjs` で起動し、`HOST` / `PORT` で待ち受けを調整します。
