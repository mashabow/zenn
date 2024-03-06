---
title: "ローディング状態の表示を VRT で手軽に担保する"
emoji: "⏳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vrt", "storybook", "msw", "regsuit", "react"]
published: true
publication_name: "socialplus"
---

こんにちは、mashabow です。[昨日](https://zenn.dev/socialplus/articles/c3b2e14b9087bc)から始まった [Social PLUS Tech Blog](https://zenn.dev/p/socialplus) の技術記事 1 本目なのでビビっています 🤗

## UI のローディング状態

さて、ローディング状態ってありますよね。以下はソーシャル PLUS の画面の一例ですが、API からのデータ取得中に表示する、こういうやつです。

![ローディング状態の表示の例](https://storage.googleapis.com/zenn-user-upload/07cad4fa4565-20240124.png)

最近は[スケルトンスクリーン](https://note.com/golferdayo/n/na8f69dc61f8e)を表示することも当たり前になっています。細かな気遣いですが、ローディング状態は快適な UX のためには欠かせません。

しかしこのローディング状態、普段コードを書いている最中は、ローカルの API モックを使ったり高速なネットワーク環境があったりするので、ともすればうっかり見過ごしがちな部分です。ローディング状態の表示が壊れていたときに、はたして気づけるでしょうか？ ブラウザの DevTools で[ネットワークのスロットリング](https://developer.chrome.com/docs/devtools/network?hl=ja#throttle)を有効にすれば確認はできますが、いちいち気にするのは面倒です。そして、スロットリングを解除し忘れて自分にイラッとするのもあるあるです（ですよね？）。

ということで、この記事では **「ローディング状態の表示が壊れていないこと」をできるだけ手軽に確認・担保する** 仕組みをチュートリアル形式で紹介します。なお、「ローディング状態の表示」だとちょっと長いので、以下では単に「ローディング表示」と呼ぶことにします。

## Storybook でローディング表示を確認できるようにする

まずは Storybook でローディング表示を確認できるとうれしいですよね？ 「このコンポーネントってどんなローディング表示だったっけ？」と思ったときに、ローディング表示の story があれば、エンジニアもデザイナーも手軽に確認することができます。

では、どう書けばいいでしょうか？ もちろん、コンポーネントに `isLoading` のような prop があれば簡単です。

```tsx:Foo.stories.tsx
export const Loading: Story = {
  args: {
    isLoading: true,
  },
};
```

が、すべてのコンポーネントがそんな都合のいい設計になっているわけでもありません。コンポーネントの中でリクエストを飛ばす場合も普通にあるでしょう。ページコンポーネントのような、粒度の大きいコンポーネントであればなおさらです。

そこで、[MSW](https://mswjs.io/) を使って「レスポンス待ち」の状態を再現することを考えます。MSW では、[`ctx.delay('infinite')`](https://v1.mswjs.io/docs/api/context/delay)（v1）や [`delay('infinite')`](https://mswjs.io/docs/api/delay/#delay-modes)（v2）を使うと「いつまでたってもレスポンスが返ってこない[リクエストハンドラ](https://v1.mswjs.io/docs/basics/request-handler)」を作ることができます。

```ts
import { rest } from "msw";

rest.get("/foo", (_req, res, ctx) => res(ctx.delay("infinite")));
```

:::message
昨年 MSW v2 がリリースされていますが、[`msw-storybook-addon`](https://storybook.js.org/addons/msw-storybook-addon) の安定版が[まだ v2 に対応していない](https://github.com/mswjs/msw-storybook-addon/issues/121)ため、この記事では MSW v1 を前提に説明をしています。とはいえ、v2 でも同じことができるはずです。
:::

まず、MSW と [`msw-storybook-addon`](https://storybook.js.org/addons/msw-storybook-addon) を導入します。導入手順についてはこの記事では割愛しますので、公式のドキュメントを参照してください。次に、いつまで経ってもレスポンスが返ってこないリクエストハンドラを作り、それを使った `Loading` story を用意します。

```tsx:Foo.stories.tsx
import { rest } from "msw";

export const Loading: Story = {
  parameters: {
    msw: [
      rest.get(
        "https://example.com/foo", // Foo コンポーネント内で叩いているエンドポイント
        (_req, res, ctx) => res(ctx.delay("infinite")),
      ),
    ],
  },
};
```

これでローディング表示の story ができました 🎉

## もう少し楽にリクエストハンドラを書く

でも、リクエスト先のエンドポイントの数だけリクエストハンドラを書くのは面倒ですよね？ また、例えばページコンポーネントの `Loading` story を書こうとした場合、「子や孫のコンポーネントがどこにリクエストしているか」なんて気にしたくありません。

というわけで、楽する方法を考えます。簡単のために「`Loading` story では、どこに対するリクエストであってもレスポンスが返ってこない」という前提を置くことにしましょう。リクエストハンドラの URL 指定には[正規表現が使える](https://v1.mswjs.io/docs/basics/request-matching#regexp)ので、雑に `/.*/` を指定します。HTTP メソッドも `GET` が多いとは思いますが、この際なのですべてのメソッドを対象にする [`rest.all()`](https://v1.mswjs.io/docs/api/rest#custom-methods) を使いましょう。これで、**どんなリクエストに対しても永遠にレスポンスを返さないリクエストハンドラ**である `infiniteRequestHandler` ができました。

```ts:handlers.ts
import { rest } from "msw";

export const infiniteRequestHandler = rest.all(/.*/, (_req, res, ctx) =>
  res(ctx.delay("infinite"))
);
```

あとは `Loading` story に指定するだけです。

```tsx:Foo.stories.tsx
import { infiniteRequestHandler } from "./handlers"

export const Loading: Story = {
  parameters: {
    msw: [infiniteRequestHandler],
  },
};
```

上のように `infiniteRequestHandler` を別ファイルに切り出しておけば、**いろいろな `*.stories.tsx` ファイルからインポートして使い回すことができます**。やっていることは単純ですが、だいぶ楽になりました 👍

状況によっては「このリクエストは `infiniteRequestHandler` から除外したい」ようなケースがあるかもしれませんが、そのような場合は個別に上書きしてやれば OK です。

```tsx:Foo.stories.tsx
export const PartiallyLoading: Story = {
  parameters: {
    msw: [
      infiniteRequestHandler,
      // 個別に上書き
      rest.get(
        "https://example.com/foo",
        (_req, res, ctx) => res(ctx.json({ message: "Hi!" })),
      ),
    ],
  },
};
```

## ローディング表示が壊れていないことを VRT で担保する

ローディング表示の story を用意しましたが、これだけでは「開発の途中でいつのまにかローディング表示が壊れていた」ということが起こりえます。そこで、「ローディング表示が壊れていないこと」を担保する方法について考えていきます。これについても**できるだけ手軽に**やりたいので、ビジュアルリグレッションテスト（VRT）を活用することにしましょう。

:::message
この記事では VRT の概念そのものや、各ツールのセットアップ方法については説明を割愛します。公式ドキュメントなどを参照してください。『[フロントエンド開発のためのテスト入門](https://amzn.to/3SdPbdW)』もおすすめです。
:::

定番の構成ですが、先ほど用意した `Loading` story のスクリーンショットを [Storycap](https://github.com/reg-viz/storycap) で撮影し、[reg-suit](https://reg-viz.github.io/reg-suit/) を使って前のコミットのスクリーンショットと比較することにします。

しかし、そのまま Storycap を実行するとスクリーンショットの撮影が終わらず、タイムアウトしてしまいます。というのも、Storycap のデフォルトでは「リクエストがすべて完了するまで待ってから、スクリーンショットを撮影する」という制御が入っているからです。この待ち処理はスクリーンショットを安定させるためのものですが、Fetch API によるリクエストも対象になっています。そのため、`infiniteRequestHandler` を使っている story では、いつまで経ってもスクリーンショットの撮影が行われません。

このタイムアウトを回避するためには、**`waitAssets` オプションと `waitImages` オプションを両方とも `false` にして、リクエスト待ちを無効にする必要があります**。

```tsx:Foo.stories.tsx
import { infiniteRequestHandler } from "./handlers"

export const Loading: Story = {
  parameters: {
    msw: [infiniteRequestHandler],
    // Storycap の設定
    screenshot: {
      waitAssets: false,
      waitImages: false,
    },
  },
};
```

これでタイムアウトせずに、無事スクリーンショットを撮影できるようになりました！

:::details Storycap の待ち処理の詳細
この待ち処理は、`ResourceWatcher` クラスの以下の箇所で行われています。各リクエストに対して resolve されたかどうかをチェックしているようです。

https://github.com/reg-viz/storycap/blob/v4.2.0/packages/storycrawler/src/browser/resource-watcher.ts#L93-L107

https://github.com/reg-viz/storycap/blob/v4.2.0/packages/storycrawler/src/browser/resource-watcher.ts#L52-L67

`fetch` などによるリクエスト（`request.resourceType() === 'xmlhttprequest'`）はこのチェック対象になっています。

`waitAssets` と `waitImages` の両方を `false` にすると、以下の `if` 文によって `ResourceWatcher` による待ち処理がスキップされます。

https://github.com/reg-viz/storycap/blob/v4.2.0/packages/storycap/src/node/capturing-browser.ts#L314-L318

:::

## `waitAssets` と `waitImages` オプションを自動で設定する

さて、最後にもうちょっと楽をしましょう。`infiniteRequestHandler` を使っている story に毎回毎回 `waitAssets` と `waitImages` を指定する必要があるのは、ぶっちゃけ面倒ですよね？ わたしは面倒です。指定をたびたび忘れましたし、他のメンバーが書いた `Loading` story をレビューするときにも見落とします。ということで、自動で設定してくれるようにしましょう。

全 story に対して、「`infiniteRequestHandler` を使っていたら `waitAssets` と `waitImages` を `false` にする」が自動でできれば OK なわけです。[グローバルなデコレータ](https://storybook.js.org/docs/writing-stories/decorators#global-decorators)を書きましょう。

```tsx:preview.tsx
import type { Preview } from "@storybook/react";

import { infiniteRequestHandler } from './handlers';

const preview: Preview = {
  decorators: [
    (storyFn, context) => {
      if (context.parameters.msw.includes(infiniteRequestHandler)) {
        context.parameters.screenshot.waitAssets = false;
        context.parameters.screenshot.waitImages = false;
      }
      return storyFn(context);
    },
    ...,
  ],
  ...,
};

export default preview;
```

こうすれば、`waitAssets` と `waitImages` の指定が不要になります 🎉

```tsx:Foo.stories.tsx
import { infiniteRequestHandler } from "./handlers"

export const Loading: Story = {
  parameters: {
    msw: [infiniteRequestHandler],
    // いちいち指定しなくてもよくなった
    // screenshot: {
    //   waitAssets: false,
    //   waitImages: false,
    // },
  },
};
```

## まとめ

というわけで、最終的には以下の数行だけで `Loading` story が書けるようになりました。めんどくさがりやなわたしでも、これぐらいだったらいろいろなコンポーネントに `Loading` story をビシバシ書いていけます。

```tsx:Foo.stories.tsx
import { infiniteRequestHandler } from "./handlers"

export const Loading: Story = {
  parameters: {
    msw: [infiniteRequestHandler],
  },
};
```

実際の業務でもこんな感じに `Loading` story を用意して、VRT を行っています。

![Storybook 上にたくさんある `Loading` story](https://storage.googleapis.com/zenn-user-upload/d239ce819ee7-20240124.png)

もっと便利な方法があるよ！という方がいましたら、ぜひコメントで教えてください。ではでは。
