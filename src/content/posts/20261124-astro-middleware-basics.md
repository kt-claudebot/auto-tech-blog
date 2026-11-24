---
title: "Astro ミドルウェア入門：onRequest・locals・sequence・rewrite の使い方"
description: "Astro 公式ドキュメントに沿って、ミドルウェアの配置場所、onRequest と context.locals、defineMiddleware による型付け、sequence() での連結、rewrite、エラーページでの挙動をまとめます。"
pubDate: 2026-11-24
tags: ["astro", "middleware", "typescript"]
sourceUrl: "https://docs.astro.build/en/guides/middleware/"
---

Astro 公式のミドルウェアガイドの内容を整理します。以下は公式ドキュメントに記載のある内容のみをまとめたもので、コードはドキュメントの例を元にしています（WebFetch による要約を経由しているため、正確な全文は公式を参照してください）。

## ミドルウェアとは

リクエストとレスポンスを横取りし、ページやエンドポイントがレンダリングされる直前に処理を挟む仕組みです。プリレンダリングされるページではビルド時に、オンデマンドのページではリクエスト時に実行されます（Cookie やヘッダーが使えるのは後者）。リクエスト固有のデータを `locals` に載せることもできます。

## ファイルの置き場所

次のいずれかを作成します。

- `src/middleware.js|ts`
- `src/middleware/index.js|ts`

## 基本: onRequest()

`context` と `next` を受け取る `onRequest()` を **名前付きエクスポート** します（default export ではありません）。`Response` か `next()` の結果を返します。

```js
export function onRequest (context, next) {
    // intercept data from a request
    // optionally, modify the properties in `locals`
    context.locals.title = "New title";
    context.locals.property = "information";

    // return a Response or the result of calling `next()`
    return next();
};
```

`.astro` ファイルからは `Astro.locals` で読めます。

```astro
---
const data = Astro.locals;
---
<h1>{data.title}</h1>
<p>This {data.property} is from middleware.</p>
```

## context.locals の性質

- ミドルウェア、API ルート、`.astro` ページの間で共有されます。
- 関数や Map を含む任意の型の値を持てます。
- 寿命は1回のルートレンダリングです。リクエストをまたいで残したいデータには別の保存先が必要です。
- インテグレーションが `locals` にプロパティを設定する場合があるため、上書きする前にそのドキュメントを確認してください。
- 実行時に `locals` 自体を上書きしようとするとエラーになります。

## レスポンスを加工する

`next()` を await してレスポンスの HTML を読み、加工した新しい `Response` を返せます。公式の例（機密情報のマスク）です。

```js
export const onRequest = async (context, next) => {
    const response = await next();
    const html = await response.text();
    const redactedHtml = html.replaceAll("PRIVATE INFO", "REDACTED");

    return new Response(redactedHtml, {
        status: 200,
        headers: response.headers
    });
};
```

## TypeScript での型付け

`astro:middleware` の `defineMiddleware()` を使うと `context` と `next` の型が自動で付きます。

```ts
import { defineMiddleware } from "astro:middleware";

// `context` and `next` are automatically typed
export const onRequest = defineMiddleware((context, next) => {

});
```

`Astro.locals` の型は `src/env.d.ts` でグローバルの `App.Locals` を拡張して定義します。

```ts
type User = {
  id: number;
  name: string;
};

declare namespace App {
  interface Locals {
    user: User;
    welcomeTitle: () => string;
    orders: Map<string, object>;
    session: import("./lib/server/session").Session | null;
  }
}
```

## sequence() で連結する

複数のミドルウェアを指定した順に実行します。

```js
import { sequence } from "astro:middleware";

async function validation(_, next) {
    console.log("validation request");
    const response = await next();
    console.log("validation response");
    return response;
}

async function auth(_, next) {
    console.log("auth request");
    const response = await next();
    console.log("auth response");
    return response;
}

async function greeting(_, next) {
    console.log("greeting request");
    const response = await next();
    console.log("greeting response");
    return response;
}

export const onRequest = sequence(validation, auth, greeting);
```

ログの順序は、リクエスト側が `validation → auth → greeting`、レスポンス側が `greeting → auth → validation` になります。

## Rewrite（astro@4.13.0 で追加）

`context.rewrite()` は、訪問者をリダイレクトせずに別のルートの内容を表示します。新しいレンダリングのためミドルウェアが再実行されます。

```js
import { isLoggedIn } from "~/auth.js"

export function onRequest (context, next) {
  if (!isLoggedIn(context)) {
    // If the user is not logged in, update the Request to render the `/login` route and
    // add header to indicate where the user should be sent after a successful login.
    // Re-execute middleware.
    return context.rewrite(new Request("/login", {
      headers: {
        "x-redirect-to": context.url.pathname
      }
    }));
  }

  return next();
};
```

`next()` にパス（文字列・URL・`Request`）を渡す方法もあります。こちらは新たなレンダリングを始めないため、ミドルウェアは再実行されません。`sequence()` 内では、後続のミドルウェアが更新された `context` を受け取ります。

**注意:** パス付きの `next()` は古い `ctx.request` から新しい `Request` を作ります。その前後で `Request.body` を読むとランタイムエラーになり、HTML フォームでの Astro Actions で起きやすいとされています。その場合はミドルウェアではなくテンプレート内で `Astro.rewrite()` を使うよう、公式は案内しています。

## エラーページでの挙動

- オンデマンドのページでは、ルートが一致しない場合（既定・カスタムの 404 を含む）でもミドルウェアが実行されます。ただしアダプターがプラットフォーム固有のエラーページを返す場合があります。
- 500 ページ（カスタム含む）の前にもミドルウェアは実行されますが、エラーの原因がミドルウェア自身だった場合は、500 ページのレンダリング時に `Astro.locals` が使えないことがあります。

## まとめ

- `src/middleware.ts` に `onRequest` を名前付きエクスポートする。
- リクエスト単位のデータは `context.locals` に載せ、`App.Locals` で型を付ける。
- 複数処理は `sequence()` で連結、表示の差し替えは `rewrite` を使う。

なお、このブログは静的出力のため、ミドルウェアが request ごとに動くのはオンデマンド（SSR）構成にした場合です。詳細はサイト構成に応じて公式を確認してください。
