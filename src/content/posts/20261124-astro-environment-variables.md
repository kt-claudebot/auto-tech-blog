---
title: "Astro の環境変数入門：PUBLIC_ の扱い・.env のモード・astro:env での型安全な管理"
description: "Astro 公式ドキュメントに沿って、import.meta.env と PUBLIC_ プレフィックス、.env ファイルとモード、設定ファイルでの loadEnv、astro:env による型安全な環境変数スキーマまでを整理します。"
pubDate: 2026-11-24
tags: ["astro", "environment-variables", "typescript"]
sourceUrl: "https://docs.astro.build/en/guides/environment-variables/"
---

Astro 公式の環境変数ガイドの内容を整理します。以下は公式ドキュメントに記載のある内容のみをまとめたもので、コードはドキュメントの例を元にしています（WebFetch による要約を経由しているため、正確な全文は公式を参照してください）。

## import.meta.env と PUBLIC_ プレフィックス

Astro は Vite の環境変数サポートを使っており、値はビルド時に静的に置き換えられます。読み取りには `process.env` ではなく `import.meta.env` を使います。

- サーバー側のコードはすべての変数にアクセスできます。
- クライアント側のコードが見られるのは、`PUBLIC_` プレフィックスの付いた変数だけです（セキュリティ上の措置）。

`.env` の例です。

```
SECRET_PASSWORD=password123
PUBLIC_ANYBODY=there
```

`PUBLIC_ANYBODY` はサーバー・クライアントの両方で使え、`SECRET_PASSWORD` はサーバー専用です。

## .env ファイルとモード

- 変数はプロジェクトディレクトリの `.env` ファイルから読み込まれます。`.env.testing` や `.env.staging` のようなモード別ファイルも使えます。
- `astro dev` は既定で `development` モード、`astro build` は `production` モードで動きます。
- `--mode` フラグで別のモードと対応する `.env` ファイルを選べます。

```
npm run astro dev -- --mode staging
npm run astro build -- --mode testing
```

- Astro はまずサーバーの環境変数を確認し、なければ `.env` ファイルにフォールバックします。
- CLI からインラインで渡すこともできます（例: `PUBLIC_POKEAPI=https://pokeapi.co/api/v2 npm run dev`）。

## 組み込みの変数

- `import.meta.env.MODE`: `astro dev` では `development`、`astro build` では `production`。
- `import.meta.env.PROD` / `import.meta.env.DEV`: 常に互いに逆の真偽値。
- `import.meta.env.BASE_URL`: `base` 設定で決まる。
- `import.meta.env.SITE`: `site` 設定で決まる。

```
const isProd = import.meta.env.PROD;
```

## TypeScript の補完（env.d.ts）

`import.meta.env` は `astro/client.d.ts` で既定の型が付いています。独自の変数にも補完を効かせたい場合は、`src/env.d.ts` を作って `ImportMetaEnv` と `ImportMeta` を拡張します。公式の例では `readonly DB_PASSWORD: string;` や `readonly PUBLIC_POKEAPI: string;` のようなフィールドを宣言しています。

## 設定ファイルでは .env が読まれない

Astro は設定ファイルを他より先に評価するため、設定ファイル内では `.env` ファイルは読み込まれません。

- `astro.config.mjs` では、CLI 経由で設定した変数を `process.env` から読めます。
- `.env` を手動で読みたい場合は Vite の `loadEnv` を使います。

```
const { SECRET_PASSWORD } = loadEnv(process.env.NODE_ENV, process.cwd(), "");
```

pnpm を使う場合、`loadEnv` を import するには `vite` を直接インストールする必要があります。

SSR の実行時は、多くのアダプターが `process.env` を使います。Deno アダプターは `Deno.env.get()` を使い、Cloudflare アダプターには独自のランタイムアクセス方法があります。

## astro:env で型安全に扱う

`astro.config` の `env.schema` に `envField` で変数を定義します。各変数には `context`（`"client"` か `"server"`）と `access`（`"secret"` か `"public"`）が必須で、`optional` や `default` は任意です。

```
API_SECRET: envField.string({ context: "server", access: "secret" }),
```

データ型は `envField.string`、`envField.number`、`envField.boolean`、`envField.enum`（`values` 配列を取る）が使えます。

変数の種類と import 元は次のとおりです。

- 公開クライアント変数: クライアント・サーバー両方のバンドルに含まれ、`astro:env/client` から import。
- 公開サーバー変数: サーバーバンドルに含まれ、`astro:env/server` から import。
- シークレットのサーバー変数: バンドルされず、`astro:env/server` から import。シークレットのクライアント変数はサポートされません。
- シークレットは `astro:env/server` から何かを import した時点で検証されます。`validateSecrets` 設定で起動時に検証することも可能です。

Astro コンポーネント、エンドポイント、ミドルウェア、フレームワークコンポーネントから利用できます。`astro.config.mjs` や素のスクリプトでは使えず、その場合は `process.env` を使います。

スキーマに無い値や生のシークレットは `getSecret` で取得できます。

```
getSecret("FOO"); // string | undefined
```

## まとめ

- クライアントに出したい値だけ `PUBLIC_` を付ける。それ以外はサーバー専用。
- モード別の `.env.<mode>` は `--mode` で切り替える。
- 設定ファイルでは `.env` が自動で読まれないので `loadEnv` を使う。
- 型と検証が欲しければ `env.schema` と `astro:env` を使う。

なお、`astro:env` が導入された Astro のバージョンは参照した公式ページからは確認できていません。利用する Astro のバージョンで使えるかは公式で確認してください。関連: [Astro ミドルウェア入門](/posts/20261124-astro-middleware-basics/)。
