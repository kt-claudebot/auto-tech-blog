---
title: "Astro Actions 入門：型安全なサーバー関数の定義・呼び出し・エラー処理・セキュリティ"
description: "Astro 公式ガイドに沿って、Astro Actions の定義方法（defineAction と Zod）、クライアントからの呼び出し、ActionError と入力エラー処理、フォーム連携、セキュリティ上の注意点を整理します。"
pubDate: 2026-11-24
tags: ["astro", "actions", "typescript"]
sourceUrl: "https://docs.astro.build/en/guides/actions/"
---

Astro 公式の Actions ガイドの内容を整理します。以下は公式ドキュメントに記載のある内容のみをまとめたものです。WebFetch による要約を経由しており、コードは要約に含まれた最小構成の例です。正確な全文・詳細な例は公式を参照してください。

## Astro Actions とは

Actions は、データ取得・JSON のパース・入力検証を扱う型安全なバックエンド関数です。多くの API エンドポイントや手書きの `fetch()` 呼び出しの代わりに使えます。

## アクションの定義

- `src/actions/index.ts` から `server` オブジェクトをエクスポートします。
- 各アクションは `defineAction()` で作り、Zod（`astro/zod` から import する `z`）による `input` スキーマと、サーバーで実行される `handler` を持ちます。
- `user` のようなオブジェクトにまとめて別ファイルに分け、メインのファイルで import することもできます。

```ts
export const server = {
  getGreeting: defineAction({
    input: z.object({ name: z.string() }),
    handler: async (input) => `Hello, ${input.name}!`,
  }),
};
```

（上記は import 文を省略した断片です。`z` は `astro/zod` から取得します。）

## クライアントからの呼び出し

- UI フレームワークのコンポーネントや Astro の `<script>` タグ内で、`astro:actions` から `actions` を import して呼び出します。
- 呼び出しは `{ data, error }` を返します。まず `error` を確認するのが推奨されるパターンです。
- `.orThrow()` は `data` を直接返し、エラー時は例外を投げます。プロトタイピング向きです。

## エラー処理

- `ActionError` を `code`（例: `UNAUTHORIZED`）と任意の `message` 付きで throw すると、レコードが存在しない・未認証といった失敗を表現できます。
- クライアントは `error.code` で分岐し、状況に応じた UI を出せます。
- `isInputError()` でバリデーション失敗を判別でき、`error.fields` に入力名ごとのメッセージが入ります。
- 成功後のクライアント側リダイレクトには Astro の `navigate()` が使えます。

## フォーム入力

- `defineAction()` に `accept: 'form'` を指定すると `FormData` を受け取れます。
- Zod のバリデーターがフィールドに適用され、数値・チェックボックス・ファイル・同名の繰り返しフィールドには特別な扱いがあります。
- discriminated union を使うと、1 つのフォームで複数の形を扱えます。

## HTML フォームからの利用

- `<form>` に `method="POST"` と `action={actions.name}` を指定します。クライアント側 JavaScript なしでも動作します。
- フォームアクションを使うページは、プリレンダリングではなくオンデマンドレンダリングである必要があります。
- サーバー側では `Astro.getActionResult(actions.name)` で結果を取得し、リダイレクトやエラー表示、UI 更新に使えます。
- ミドルウェアで `getActionContext()` を使えば、POST/Redirect/GET パターンでリクエスト間に結果を保持できます。

## サーバーコードからの呼び出し

- Astro コンポーネントでは `Astro.callAction(actions.name, input)` を使います。
- サーバーエンドポイントでは `context.callAction(...)` を使います。
- クライアント呼び出しと同じ `data` / `error` 形式の結果が返ります。

## セキュリティ上の注意

- アクションは名前で公開到達可能です（例: `/_actions/` 配下）。API エンドポイントと同様の認可チェックが必要です。
- ハンドラー内で `context.locals` などを参照して権限を確認し、`UNAUTHORIZED` の `ActionError` を投げます。
- ミドルウェアで `action.calledFrom === 'rpc'` を確認して RPC 形式の呼び出しを制限することもできますが、適切な認可の代わりにはなりません。

## 関連記事

ミドルウェアと組み合わせる場面が多いため、あわせて [Astro のミドルウェア入門](/posts/20261124-astro-middleware-basics/) や [環境変数入門](/posts/20261124-astro-environment-variables/) も参照してください。
