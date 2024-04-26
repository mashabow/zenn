---
title: "モノレポ内で Shopify CLI を使ったら依存パッケージのロックが効いていなかった"
emoji: "😱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [shopify, monorepo, yarn]
published: true
publication_name: "socialplus"
---

こんにちは、mashabow です。[前回は自慢話](https://zenn.dev/socialplus/articles/bug-bounty-money)だったので、今回は失敗談を紹介してバランスをとっておこうかと思います。モノレポ + Shopify CLI にまつわる事故です。

去年の話になりますが、「とある入力欄のプレースホルダに `例）` を付け足す」という、ごく軽微な改善を行いました。

![](https://storage.googleapis.com/zenn-user-upload/94d34c4d0ee8-20240430.png =350x)
_`例）` を付け足してプレースホルダだということをわかりやすくした_

このとき変更したのはこの 1 箇所だけです。さくっとマージしてｼｭｯと本番デプロイして気持ちよく次の仕事に取りかかっていたところ……アプリの別の機能が止まった、というアラートが飛んできました 😱

## 当時の状況

ソーシャル PLUS 社内のフロントエンド関連のコードは、その大半を Yarn workspaces + [Turborepo](https://turbo.build/repo) のモノレポとして管理しています。このモノレポには、Shopify アプリ「CRM PLUS on LINE」のフロントエンド部分も含まれます。

https://crmplus.socialplus.jp/

Shopify アプリには [app extensions](https://shopify.dev/docs/apps/structure/app-extensions) と呼ばれる仕組みがあります。Shopify アプリは app extensions を利用して Shopify の UI へと入り込み、さまざまな機能を提供しています。CRM PLUS on LINE も、[theme app extension](https://shopify.dev/docs/apps/online-store/theme-app-extensions) や [Web pixels](https://shopify.dev/docs/apps/marketing/pixels) などの app extensions を利用しており、これらは Shopify CLI によって管理されていました[^1]。

[^1]: ちなみに、CRM PLUS on LINE の埋め込みアプリ（embedded app）部分の開発には Shopify CLI は利用していません。埋め込みアプリの開発を始めた当初は Shopify CLI が今ほど安定・充実していなかったから、という経緯です。

https://shopify.dev/docs/apps/tools/cli

リポジトリ全体としては、ざっくり以下のような構成です。

```txt
├── apps/
│   ├── crm-plus-on-line-extensions/
│   │   ├── extensions/
│   │   │   ├── foo-extension/
│   │   │   └── bar-extension/
│   │   └── package.json  👈 Shopify CLI はここで使われる
│   └── …
├── package.json  👈 Yarn workspaces のルート
└── yarn.lock
```

`apps/crm-plus-on-line-extensions/` は、`$ shopify app init` で scaffold したものがベースになっています。このディレクトリがひとつの Yarn workspace です。Shopify CLI はこの workspace でしか使わないため、この workspace の `package.json` の `dependencies` に指定しています。

```js:apps/crm-plus-on-line-extensions/package.json
{
  // ...
  "scripts": {
    "build": "shopify app build",
    // ...
  },
  "dependencies": {
    "@shopify/cli": "3.48.4",
    // ...
  }
}
```

## 発生した問題

この workspace `apps/crm-plus-on-line-extensions/` で `$ yarn build`（`shopify app build`）を実行すると、app extensions がビルドされます。それはいいのですが、**ビルドの前に依存パッケージが npm でインストールされる**という問題がありました。

`shopify app build` は（おせっかいなことに）「ビルドする前に依存パッケージをインストールする」という仕様なのですが、このモノレポで使っている Yarn ではなく、勝手に npm でインストールを実行してしまいます。依存パッケージのバージョンを `yarn.lock` でロックしていたつもりでしたが、npm は `yarn.lock` など見ません。こうして、**`yarn.lock` とは別のバージョンがインストールされていた**のでした。

app extension 開発を最初に担当したメンバー A さんは、「なぜか npm の `package-lock.json` が作られるな 🤔」と気づいていたようですが、**`package-lock.json` を `.gitignore` に追加**してしまいました。A さんの環境には `package-lock.json` が残っているため、`shopify app build` すると npm インストールでそちらが参照され、`package-lock.json` でロックされたバージョンの依存パッケージがインストールされます。一方、別のメンバー B さんが後日 `shopify app build` する際には、もちろん手元には `package-lock.json` は存在しません。したがって、**「A さんと同じバージョンがインストールされる」という保証は全くありません**。

さらに悪いことは重なります。当時はこの app extensions の CD（継続的デリバリー）が未整備で、**ローカルマシンから手動でビルドとデプロイ**を行っていました。一応の言い訳として

- app extensions の開発を始めた当初はまだ Shopify CLI が発展途上で、`shopify app deploy` が CI/CD 上では使えなかった
- app extensions のコード量が少なく、変更を加える頻度も低かったため、CD 整備の優先度が低かった

という背景がありましたが、こういうときに限って事故が起きるものです。冒頭に書いたように、わたしが「入力欄のプレースホルダの頭に `例）` を付け足す」というめちゃくちゃ軽微な変更を行って本番環境にデプロイしたところ、app extensions の別の部分が**動かなくなってしまいました** 😱

監視アラートが飛んできてから急いでロールバックしましたが、この時点では原因は何もわかっていませんでした。上に書いた原因は、後の調査で判明したものです。

## なぜインストールに npm を使ってしまうのか？

では、Shopify CLI はなぜインストールに npm を使ってしまうのでしょうか？ 結論から言うと、どうも **Shopify CLI の Yarn workspaces 対応が部分的なものだから**なようです。同じような issue もありました。

https://github.com/Shopify/cli/issues/1895

Shopify CLI が Yarn workspaces に対応したのは、2023 年の 5 月です。

https://github.com/Shopify/cli/pull/1778

また、Shopify CLI は「Yarn が使われていたら Yarn を使う、pnpm だったら pnpm」というように、パッケージマネージャーを検出して動作を切り替えるようになっています。

https://github.com/Shopify/cli/blob/3.59.1/packages/cli-kit/src/public/node/node-package-manager.ts#L98-L122

しかし、これらの対応が不完全なようです。上の `getPackageManager()` でパッケージマネージャーを検出していますが、ざっくり以下のような判定処理になっています。

1. カレントディレクトリから親をたどって `package.json` を見つける
2. `package.json` があるディレクトリに `yarn.lock` があれば Yarn
3. `package.json` があるディレクトリに `pnpm-lock.yaml` があれば pnpm
4. `package.json` があるディレクトリに `bun.lockb` があれば Bun
5. どれもなければ npm

今回のようなディレクトリ構成の場合、`apps/crm-plus-on-line-extensions/` workspace 内には `yarn.lock` は存在しません。そのため、この workspace で Shopify CLI を実行すると、`getPackageManager()` は「npm」と誤判定してしまいます。

```txt
├── apps/
│   ├── crm-plus-on-line-extensions/
│   │   ├── extensions/
│   │   │   ├── foo-extension/
│   │   │   └── bar-extension/
│   │   └── package.json  👈 Shopify CLI はここで実行する
│   └── …
├── package.json  👈 Yarn workspaces のルート
└── yarn.lock  👈 yarn.lock はルートの方にある
```

## 回避方法

この問題の回避方法については、先ほどの issue にある [Shopify CLI 開発者のコメント](https://github.com/Shopify/cli/issues/1895#issuecomment-1559032417)が参考になります。以下に 2 つ紹介します。

### A. `--skip-dependencies-installation` をつける

`--skip-dependencies-installation` オプションをつけると、その名のとおり依存パッケージのインストールがスキップされます。`shopify app build` だけでなく `shopify app dev` にもおせっかいインストール機能があるため、同様にオプションをつけましょう。

```js:apps/crm-plus-on-line-extensions/package.json
{
  "scripts": {
    "build": "shopify app build --skip-dependencies-installation",
    "dev": "shopify app dev --skip-dependencies-installation",
    // ...
  }
}
```

こうすれば、Vite や Next.js といった「普通」のツールと同じ感触で使えます。

```sh
$ yarn install
$ yarn build  # or $ yarn dev
```

この `--skip-dependencies-installation` をつける方法は手軽ですが、"Deprecated, use workspaces instead." という非推奨の注意書きがあるのが若干気がかりといえば気がかりです。

https://github.com/Shopify/cli/blob/3.59.1/packages/cli/README.md#L70

ちなみにですが、`shopify app deploy` の方にはおせっかいインストール機能はありません。ただし、ビルド処理はついでにやってくれます。

```sh
$ yarn install
$ yarn deploy
```

### B. Yarn workspaces のルートで Shopify CLI を実行する

もうひとつは、Shopify CLI を Yarn workspaces のルートから実行する方法です。

```txt
├── apps/
│   ├── crm-plus-on-line-extensions/
│   │   ├── extensions/
│   │   │   ├── foo-extension/
│   │   │   └── bar-extension/
│   │   └── package.json  👈 こっちじゃなくて…
│   └── …
├── package.json  👈 Yarn workspaces のルートから Shopify CLI を実行する
└── yarn.lock
```

```js:（ルートの）package.json
{
  // ...
  "scripts": {
    "build": "shopify app build --path app/crm-plus-on-line-extensions/",
    // ...
  },
  "dependencies": {
    "@shopify/cli": "3.48.4",
    // ...
  }
}
```

上で紹介した [Shopify CLI の Yarn workspaces 対応](https://github.com/Shopify/cli/pull/1778)というのは、もともとこのような構成（ルートから実行する構成）を想定して実装された機能のようです。

ただし、わたしたちのモノレポの場合、Shopify CLI が必要なのは `apps/crm-plus-on-line-extensions/` workspace だけです。他の workspace は Shopify CLI には全く関係がないため、Shopify CLI をルートに持ってくるのはちょっと気持ち悪いですね。

## おわりに

ということで、モノレポ + Shopify CLI で穴を踏み抜いてしまった事例を紹介しました。穴と言っても、結局のところ自分たち使い方のが悪かったんですが。たまにしかデプロイしない~~とはいえ~~**からこそ**、デプロイフローの整備は重要ですね、という教訓でした。

![](https://storage.googleapis.com/zenn-user-upload/54565517efd2-20240430.png)
_本件の障害レポートの末尾部分_

ちなみにこの事故を起こした後、CircleCI からデプロイできるように CD のフローを整備しました。それ以来、問題は起きていません 😌
