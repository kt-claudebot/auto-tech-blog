---
title: "Astro の View Transitions 入門：ClientRouter の有効化・transition ディレクティブ・注意点"
description: "Astro 公式ドキュメントに沿って、ClientRouter によるページ遷移アニメーションの有効化、transition:name / animate / persist、ライフサイクルイベント、フォールバックとスクリプト周りの注意点をまとめます。"
pubDate: 2026-11-24
tags: ["astro", "view-transitions", "ux"]
sourceUrl: "https://docs.astro.build/en/guides/view-transitions/"
---

Astro 公式の View Transitions ガイドの内容を整理します。以下は公式ドキュメントに記載のある内容のみをまとめたもので、コードは要点を示す最小の断片です（ドキュメントの要約を元にしているため、正確な全文は公式を参照してください）。

## 有効化する

Astro はデフォルトでは通常のフルページ遷移です。`astro:transitions` から `ClientRouter` を import し、共有する `<head>`（レイアウト）に `<ClientRouter />` を置くと有効になります。デフォルトのアニメーションだけなら他の設定は不要です。

```astro
---
import { ClientRouter } from "astro:transitions";
---
<head>
  <ClientRouter />
</head>
```

## transition ディレクティブ

Astro は遷移前後のページで対応する要素を、要素の種類と DOM 上の位置から自動で対応付けます。

- **`transition:name`**: 明示的に対応付けます。両ページで同じ名前を付けます（例: `<aside transition:name="hero">`）。1ページ内で同じ名前は1回だけ使えます。
- **`transition:animate`**: アニメーションを上書きします。組み込みは `fade`（デフォルトのクロスフェード）、`initial`（ブラウザ標準）、`slide`（旧コンテンツが左へ退場し新コンテンツが右から入る。戻る操作では逆）、`none`（無効化）です。`<html>` に `none` を付けるとサイト全体のデフォルトフェードを無効にでき、`<main>` にだけ `slide` を付ける、といった組み合わせも可能です。
- 組み込みアニメーションは import してオプションを渡せます（公式の例: `transition:animate={fade({ duration: "0.4s" })}`）。独自アニメーションはグローバルスタイルの `@keyframes` で `forwards` / `backwards` を定義します。
- **`transition:persist`**: 遷移をまたいで要素やアイランドを保持します（例: `<video controls muted autoplay transition:persist>`、`<Counter client:load transition:persist initialCount={5} />`）。名前を渡す `transition:persist="media-player"` の形もあります。`transition:persist-props`（astro@4.5.0 で追加）は、新しい props で再描画せず既存の props を保ちます。

## ルーターの制御

- `<a>` や `<form>` に `data-astro-reload` を付けると、そのリンクだけフルページ遷移になります。
- `data-astro-history` には `"auto"` / `"push"` / `"replace"` を指定でき、履歴の積み方を制御します。
- スクリプトやクライアントフレームワークのコンポーネントからは、`astro:transitions/client` の `navigate()` で遷移できます。

## ライフサイクルイベント

`document` 上で次の順に発火します。

1. `astro:before-preparation`: 読み込み前。ローダーや遷移方向のカスタマイズが可能。
2. `astro:after-preparation`: 新しいコンテンツの読み込み・パース完了。
3. `astro:before-swap`: DOM 入れ替え前。`event.newDocument` の変更やスワップ関数の差し替えが可能。
4. `astro:after-swap`: 入れ替え直後、描画前。
5. `astro:page-load`: 遷移完了。イベントリスナーの再設定に使います。

## フォールバック

View Transitions 非対応ブラウザ向けに `fallback` を選べます。`animate`（デフォルト・推奨）は属性で遷移を擬似的に再現、`swap` はアニメーションなしで即座に差し替え、`none` はフルページ遷移を使います。`<ClientRouter fallback="swap" />` のように指定します。なお `initial` アニメーションは擬似再現されないため、その要素はアニメーションしません。

## prefers-reduced-motion

ルーターには、モーション低減が要求された場合にフォールバックを含むすべての遷移アニメーションを無効化するメディアクエリが含まれています。この場合、ブラウザは単にコンテンツを入れ替えます。

## 注意点

- **スクリプト**: バンドルされたモジュールスクリプトは1回しか実行されません。インラインスクリプトは再実行される場合があり、`data-astro-rerun` で再実行を強制できます。遷移ごとに実行したい初期化処理は `astro:page-load` の中に書きます。
- **persist の限界**: `transition:persist` を付けても、CSS アニメーションは再開し、iframe は再読み込みされます。
- **`navigate()` とユーザー入力**: `?redirect` のようなユーザー由来の URL を渡す前に、許可リストで検証してください。
- **サーバーサイド**: サーバーレンダリング中の `navigate()` は効果がありません。
- **アクセシビリティ**: ルート読み上げはまず `<title>` を使うため、各ページに `<title>` を必ず入れます。
- **将来性**: 公式は、ブラウザ API の成熟に伴い `<ClientRouter />` の必要性は下がっていくと述べています。導入前にまだ必要かを確認しましょう。

## まとめ

まずは `<ClientRouter />` を1行足し、必要に応じて `transition:name` や `transition:persist` で細かく制御するのが基本の流れです。スクリプトの初期化は `astro:page-load` に寄せることが、導入時のつまずきを減らすポイントです。
