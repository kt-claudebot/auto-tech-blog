---
title: "Astro コンテンツコレクションの基本：定義・取得・ルート生成の流れ"
description: "Astro 公式ドキュメントに沿って、コンテンツコレクションの定義（glob ローダーと Zod スキーマ）、getCollection/getEntry での取得、render() と getStaticPaths によるページ生成までを整理します。"
pubDate: 2026-11-24
tags: ["astro", "typescript", "content-collections"]
sourceUrl: "https://docs.astro.build/en/guides/content-collections/"
---

Astro のコンテンツコレクションは、ブログ記事のような同種のデータを型安全に扱うための仕組みです。Astro 公式ドキュメントの内容に沿って、定義から公開ページ生成までの流れをまとめます。公式ドキュメントによると、コンテンツコレクションは `astro@2.0.0` で追加されました。

> 注: 以下のコードは公式の説明を基にした最小構成の例です。実際に使う際は公式ドキュメントのサンプルと見比べてください。

## 1. コレクションを定義する

コレクションは `src/content.config.ts`（`.js` / `.mjs` も可）で定義します。`defineCollection` は `astro:content`、`glob` ローダーは `astro/loaders`、`z` は `astro/zod` から import します。各コレクションには `loader` が必須で、`schema` は任意ですが推奨されています。

```ts
// src/content.config.ts
import { defineCollection } from 'astro:content';
import { glob } from 'astro/loaders';
import { z } from 'astro/zod';

const blog = defineCollection({
  loader: glob({ pattern: '**/*.md', base: './src/content/blog' }),
  schema: z.object({
    title: z.string(),
    pubDate: z.coerce.date(),
  }),
});

export const collections = { blog };
```

- `glob()` は base ディレクトリ配下でパターンに一致するファイルを読み込み、各エントリの `id` はファイル名から生成されます。
- `file()` ローダー（同じく `astro/loaders`）は単一の JSON / YAML / TOML ファイルからエントリを読み込みます。この場合、エントリには明示的な `id` が必要です。

## 2. コレクションを取得する

```astro
---
import { getCollection, getEntry } from 'astro:content';

const posts = await getCollection('blog');
const post = await getEntry('blog', 'hello-world');
---
```

- `getCollection()` はエントリの配列を返します。
- `getEntry()` はコレクション名と `id` で 1 件を返します。
- `getCollection()` が返す順序は決まっていないため、順序が重要なら自分でソートします。
- `getCollection()` にフィルタ関数を渡すと、たとえば `data.draft` で絞り込めます。

## 3. エントリを描画する

```astro
---
import { getEntry, render } from 'astro:content';

const entry = await getEntry('blog', 'hello-world');
if (!entry) throw new Error('Entry not found');
const { Content } = await render(entry);
---
<h1>{entry.data.title}</h1>
<Content />
```

`render()` は `Content` コンポーネントと、レンダリングされた見出し情報を返します。

## 4. 静的ルートを生成する

コレクションは `src/pages/` の外にあるため、`getStaticPaths()` を持つ動的ルートでページを生成します。

```astro
---
// src/pages/posts/[id].astro
import { getCollection, render } from 'astro:content';

export async function getStaticPaths() {
  const posts = await getCollection('blog');
  return posts.map((post) => ({
    params: { id: post.id },
    props: { post },
  }));
}

const { post } = Astro.props;
const { Content } = await render(post);
---
<h1>{post.data.title}</h1>
<Content />
```

`id` にスラッシュが含まれる場合は、ファイル名を `[...id].astro` のようにレスト引数にします。

## オンデマンドレンダリングの場合

アダプターを設定してオンデマンドレンダリングを使う場合は、`Astro.params` からスラッグを読み取り、`getEntry()` で取得し、見つからなければリダイレクトします。プロジェクトが `server` モードでない場合は `export const prerender = false;` を追加します。

## 補足

- ライブコレクション（`src/live.config.ts`、`getLiveEntry()` / `getLiveCollection()`）はオンデマンドレンダリング用のアダプターが必要です。
- エディタ向けの JSON Schema 生成は `astro@4.13.0` で追加されたと記載されています。

関連: [Astro の静的サイトを Cloudflare Workers にデプロイする手順](/posts/20261124-astro-static-site-cloudflare-workers/)
