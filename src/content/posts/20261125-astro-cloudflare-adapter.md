---
title: "Astro Cloudflare アダプター入門：導入手順・主要オプション・シークレットの扱い"
description: "Astro 公式の @astrojs/cloudflare ガイドに沿って、astro add と手動での導入方法、imageService などの主要オプション、wrangler secret の使い方、Astro 6 での注意点を整理します。"
pubDate: 2026-11-25
tags: ["astro", "cloudflare", "ssr"]
sourceUrl: "https://docs.astro.build/en/guides/integrations-guide/cloudflare/"
---

Astro 公式の Cloudflare アダプター（`@astrojs/cloudflare`）ガイドの内容を整理します。以下は公式ドキュメントに記載のある内容のみをまとめたものです。WebFetch による要約を経由しているため、細部は公式を参照してください。

オンデマンドレンダリングやサーバーアイランドなどの機能を使うには[アダプターが必要](/posts/20261124-astro-on-demand-rendering/)です。Cloudflare にデプロイする場合の選択肢が、このアダプターです。静的サイトだけを配信する手順は[Cloudflare Workers デプロイ記事](/posts/20261124-astro-static-site-cloudflare-workers/)を参照してください。

## 導入方法

### 方法1: `astro add`（推奨）

次のコマンドで `@astrojs/cloudflare` のインストールと設定ファイルの更新が一度に行われます。

```bash
npx astro add cloudflare
```

pnpm / yarn の場合は `pnpm astro add cloudflare` / `yarn astro add cloudflare` です。

### 方法2: 手動インストール

1. アダプターをインストールします。

```bash
npm install @astrojs/cloudflare
```

2. `astro.config.mjs` にアダプターを追加します。

```js
import { defineConfig } from 'astro/config';
import cloudflare from '@astrojs/cloudflare';

export default defineConfig({
  adapter: cloudflare(),
});
```

3. 任意で Wrangler の設定ファイルを作成します。省略した場合は Astro がデフォルト設定を生成します。公式の例は次のとおりです。

```jsonc
{
  "name": "my-astro-app",
  // Add your bindings here, e.g.:
  // "kv_namespaces": [{ "binding": "MY_KV", "id": "<namespace_id>" }]
}
```

## 主要なアダプターオプション

| オプション | デフォルト | 内容 |
| --- | --- | --- |
| `imageService` | `'cloudflare-binding'` | 画像サービスの選択 |
| `sessionKVBindingName` | `SESSION` | セッションに使う KV バインディング名 |
| `imagesBindingName` | `IMAGES` | `imageService` が `cloudflare-binding` のときの Images バインディング名 |
| `prerenderEnvironment` | `'workerd'` | プリレンダリング時の実行環境 |

`imageService` はオブジェクトでビルド時と実行時を別々に指定できます。

```js
import { defineConfig } from 'astro/config';
import cloudflare from '@astrojs/cloudflare';

export default defineConfig({
  adapter: cloudflare({
    imageService: { build: 'compile', runtime: 'cloudflare-binding' }
  }),
});
```

プリレンダリングされるページで `node:fs` などの Node.js API を使う場合、`workerd` が対応していないため `prerenderEnvironment` を `'node'` にします。オンデマンドのページは常に `workerd` で動作します。画像については[画像最適化の記事](/posts/20261124-astro-image-optimization/)も参照してください。

## シークレットと環境変数

シークレットは設定ファイルに書かず、Wrangler CLI で登録します。

```bash
npx wrangler secret put <KEY>
```

ローカル開発では `.dev.vars` ファイルに記述します。Astro 側の扱いは[環境変数の記事](/posts/20261124-astro-environment-variables/)も参考になります。

## その他の注意点

- アダプターは `@cloudflare/vite-plugin` の一部オプション（`persistState`、`remoteBindings` など）も受け付けます。
- セッションはデフォルトで Workers KV を使い、名前空間はデプロイ時に自動作成されると公式に記載があります。
- Astro 6 ではアダプターの v13 が必要です。Cloudflare Pages はサポート対象外となったため、デプロイ先は Cloudflare Workers にするよう公式に書かれています。

## まとめ

導入は `npx astro add cloudflare` が最短で、細かい制御が必要なら `astro.config.mjs` と Wrangler 設定を手動で整えます。Astro 6 系ではデプロイ先が Workers である点に注意してください。
