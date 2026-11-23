---
title: "Astro の静的サイトを Cloudflare Workers にデプロイする手順（wrangler.jsonc 設定つき）"
description: "Astro 公式ドキュメントに沿って、静的サイトを Cloudflare Workers にデプロイするまでの手順を整理。Wrangler の導入、wrangler.jsonc、ローカルプレビュー、Git 連携によるCI/CDまでを解説します。"
pubDate: 2026-11-24
tags: ["astro", "cloudflare", "deploy"]
sourceUrl: "https://docs.astro.build/en/guides/deploy/cloudflare/"
---

Astro で作った静的ブログを Cloudflare で公開する手順を、Astro 公式ドキュメントの内容に沿ってまとめます。以前は Cloudflare Pages が定番でしたが、公式ドキュメントでは Cloudflare Workers でのデプロイが案内されており、Cloudflare は新規プロジェクトには Workers を推奨しています。

## 前提

- Cloudflare アカウント（無料アカウントで進められます）
- Astro プロジェクト（静的出力）

静的サイトの場合、オンデマンドレンダリング用のアダプター（`npx astro add cloudflare`）は不要です。

## 1. Wrangler をインストールする

```
npm install wrangler@latest --save-dev
```

## 2. wrangler.jsonc を作成する

プロジェクトルートに `wrangler.jsonc` を作成します。静的サイトの例は次のとおりです。

```json
{
  "name": "my-astro-app",
  "compatibility_date": "YYYY-MM-DD",
  "assets": {
    "directory": "./dist"
  }
}
```

`compatibility_date` にはデプロイする日の日付を入れます（公式の例でも「デプロイする日に更新する」旨のコメントがあります）。`directory` は Astro のビルド出力先 `./dist` を指します。

## 3. ローカルでプレビューする

```
npx astro build && npx wrangler dev
```

## 4. デプロイする

```
npx astro build && npx wrangler deploy
```

アセットのアップロード後、Wrangler がプレビュー用の URL を表示します。

## Git 連携で自動デプロイする（Workers Builds）

手順 1〜3 を終えたうえで、次の流れで設定します。

1. Cloudflare ダッシュボードの Compute > Workers & Pages で「Create application」を選ぶ
2. 「Import a repository」で Git アカウントと対象リポジトリを選ぶ
3. ビルドコマンドに `npx astro build`、デプロイコマンドに `npx wrangler deploy` を設定する
4. 「Save and Deploy」を選ぶ

デプロイ後、`workers.dev` のサブドメインでサイトが公開されます。

## カスタム 404 ページ

Workers プロジェクトでカスタム 404 を返したい場合は、`assets` オブジェクト内に次を追加します。

```json
"not_found_handling": "404-page"
```

## 注意点

- ハイドレーション時に「Hydration completed but contains mismatches」というコンソールメッセージが出る場合は、Cloudflare の設定で Auto Minify を無効にしてください（公式ドキュメントの記載）。
- 既存の Cloudflare Pages プロジェクトからの移行手順は、Cloudflare の移行ガイドと互換性表を参照するよう公式ドキュメントで案内されています。

## まとめ

静的な Astro サイトの場合、必要なのは Wrangler の導入と数行の `wrangler.jsonc` だけです。独自ドメインの接続など、アカウント操作が必要な部分は Cloudflare ダッシュボードで行います。最新の手順は[公式ドキュメント](https://docs.astro.build/en/guides/deploy/cloudflare/)で確認してください。
