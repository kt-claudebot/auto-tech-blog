---
title: "Astro の画像最適化入門：Image / Picture コンポーネントと保存場所の使い分け"
description: "Astro 公式ドキュメントに沿って、src と public の画像の違い、astro:assets の Image / Picture コンポーネント、リモート画像の許可設定、Cloudflare 向けの passthrough サービスをまとめます。"
pubDate: 2026-11-24
tags: ["astro", "images", "performance"]
sourceUrl: "https://docs.astro.build/en/guides/images/"
---

Astro 公式の画像ガイドの内容を、実務で迷いやすい順に整理します。以下は公式ドキュメントに記載のある内容のみをまとめたものです。

## 画像はどこに置くか

- `src/` に置いた画像は、Astro が変換・最適化・バンドルできます。可能な限り `src/` に置くのが公式の推奨です。
- `public/` のファイルは「処理なし」でそのままビルド結果にコピーされます。加工したくない場合や、URL を固定したい場合に使います。

## Image コンポーネント

`astro:assets` から import して使います。`alt` は必須で、装飾目的の画像は `alt=""` にします。

```astro
---
import { Image } from 'astro:assets';
---
<Image src={localBirdImage} alt="A bird sitting on a nest of eggs." />
```

`public/` の画像やリモート画像は URL 文字列で指定し、`width` と `height` を渡します。

```astro
<Image src="/images/bird-in-public-folder.jpg" alt="A bird." width="50" height="50" />
<Image src="https://example.com/remote-bird.jpg" alt="A bird." width="50" height="50" />
```

レスポンシブ挙動は `layout` で有効にできます（公式の例は `layout='constrained' width={800} height={600}`）。`object-fit` / `object-position` は `fit` / `position` で上書きできます。

## Picture コンポーネント

`<Picture />` は複数フォーマットを出力し、フォールバック画像も用意します（`astro@3.3.0` で追加）。

```astro
<Picture src={myImage} formats={['avif', 'webp']} alt="A description of my image." />
```

## リモート画像の許可

リモート画像を最適化するには、`image.domains` または `image.remotePatterns` で許可します。許可されていない配信元の画像は、処理されずにそのまま表示されます。

```js
domains: ["astro.build"],
remotePatterns: [{ protocol: "https" }],
```

## 画像サービスと Cloudflare

`astro:assets` の既定の画像サービスは Sharp で、`image.service` で設定できます。公式ドキュメントは、Sharp に対応していないアダプター（Cloudflare が例として挙げられています）では、変換を行わない no-op サービスを使うよう案内しています。

```js
service: passthroughImageService()
```

`passthroughImageService` は `astro/config` から import します。この場合 Astro は画像を変換しません。

## まとめ

- 最適化したい画像は `src/` に置き、`Image` / `Picture` で配信する。
- `alt` は必須。装飾画像は空文字にする。
- リモート画像は `image.domains` / `image.remotePatterns` で許可する。
- Cloudflare などで Sharp が使えない場合は passthrough サービスを検討する。

> 注: コードは公式ドキュメントの要約から転記した断片です。設定ファイル全体の書き方は公式ガイドで確認してください。

関連: [Astro 静的サイトを Cloudflare Workers にデプロイする手順](/posts/20261124-astro-static-site-cloudflare-workers/)
