---
title: "Astro サーバーアイランド入門：server:defer・フォールバック・props の制約"
description: "Astro 公式ガイドに沿って、サーバーアイランド（server:defer）の使い方、フォールバック表示、props のシリアライズ制約、キャッシュと ASTRO_KEY の注意点を整理します。"
pubDate: 2026-11-25
tags: ["astro", "ssr", "typescript"]
sourceUrl: "https://docs.astro.build/en/guides/server-islands/"
---

Astro 公式の Server Islands ガイドの内容を整理します。以下は公式ドキュメントに記載のある内容のみをまとめたものです。WebFetch による要約を経由しているため、細部は公式を参照してください。

## サーバーアイランドとは

ページ内の動的・パーソナライズされた部分を、ページの他の部分とは別にオンデマンドで描画する仕組みです。訪問者はまずメインコンテンツを素早く見られ、静的な部分はより積極的にキャッシュできます。遅延描画を扱うために[アダプター](/posts/20261124-astro-on-demand-rendering/)が必要で、ディレクティブを使うコンポーネントはオンデマンドレンダリング環境で動かす必要があります。

## `server:defer` で有効化する

任意のコンポーネントに `server:defer` ディレクティブを付けると、そのコンポーネントが独立したアイランドになります。

```astro
---
import Avatar from '../components/Avatar.astro';
---
<Avatar server:defer />
```

コンポーネント内では、オンデマンドレンダリングのページと同様に、データ取得や Cookie の読み取りができます。

```astro
---
import { getUserAvatar } from '../sessions';
const userSession = Astro.cookies.get('session');
const avatarURL = await getUserAvatar(userSession);
---
<img alt="User avatar" src={avatarURL} />
```

## フォールバックコンテンツ

名前付きスロット `"fallback"` で、アイランドの読み込み中に表示するプレースホルダーを指定できます。フォールバックはページの他の部分と一緒に表示され、アイランドの準備ができると置き換わります。

```astro
---
import Avatar from '../components/Avatar.astro';
import GenericAvatar from '../components/GenericAvatar.astro';
---
<Avatar server:defer>
  <GenericAvatar slot="fallback" />
</Avatar>
```

## 仕組み

ビルド時に Astro は各アイランドをページから取り除き、代わりに小さなスクリプトとフォールバックを挿入します。各アイランドは独自のルートになり、ブラウザ側でスクリプトがそのルートをリクエストして HTML を受け取ります。アイランドは独立して読み込まれるため、遅いものが他をブロックしません。

## props の制約

props はシリアライズ可能である必要があります。

- 関数は渡せません。
- 循環参照を持つオブジェクトはシリアライズできません。
- 対応する型: プレーンオブジェクト、number、string、Array、Map、Set、RegExp、Date、BigInt、URL、Uint8Array、Uint16Array、Uint32Array、Infinity。

## キャッシュ

- props は暗号化されたクエリ文字列として `GET` で送られるため、標準の `Cache-Control` ヘッダーでキャッシュできます。
- ブラウザの URL 長の上限は 2048 バイトです。超えると Astro は `POST`（props は本文）に切り替えます。
- `POST` はブラウザにキャッシュされないため、既存のキャッシュ戦略が効かなくなる場合があります。必要な props だけを渡し、大きなデータは避けてください。

## ページの URL を取得する

アイランドは隔離された文脈で動くため、`Astro.url` と `Astro.request.url` は `/_server-islands/Avatar` のような URL を返します。ページ自体の情報が必要なときは、読み込み元ページの URL を持つ `Referer` ヘッダーを使います。

```astro
---
const referer = Astro.request.headers.get("Referer");
if (!referer) {
  throw new Error("Referer header is missing");
}
const url = new URL(referer);
const productId = url.searchParams.get("product");
---
```

なお、プリレンダリングされたページはクエリパラメーターを props として渡せません。

## 暗号化キー（ASTRO_KEY）

props はビルドごとにランダム生成されるキーで暗号化され、サーバーバンドルに埋め込まれます。ローリングデプロイ、マルチリージョン、アイランドを含むページを CDN でキャッシュする構成では、キーの不一致で props を復号できなくなることがあります。対策として、CLI の `astro create-key` で再利用可能なキーを生成し、環境変数 `ASTRO_KEY`（例: `.env`）に設定し、CI/CD やホストのビルド設定にも追加します。環境変数の扱いは[Astro の環境変数入門](/posts/20261124-astro-environment-variables/)も参照してください。

## まとめ

- アダプターとオンデマンドレンダリングが必須。
- props はシリアライズ可能なものだけ。長い props は `POST` になりキャッシュ不可。
- ページ URL は `Referer` から取得する。
- 複数環境・CDN 構成では `ASTRO_KEY` を固定する。

関連: [Astro Actions 入門](/posts/20261124-astro-actions-basics/)、[Astro ミドルウェア入門](/posts/20261124-astro-middleware-basics/)。
