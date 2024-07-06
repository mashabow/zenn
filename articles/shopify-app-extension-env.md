---
title: "Shopify の app extension に環境変数を埋め込む"
emoji: "🧩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [shopify, liquid]
published: true
publication_name: "socialplus"
---

この記事では、Shopify アプリを高機能にする [app extension](https://shopify.dev/docs/apps/build/app-extensions/list-of-app-extensions) に、環境変数の値を埋め込む方法をまとめます。以下、ファイル別に紹介します。

## TypeScript/JavaScript に環境変数を埋め込む

### 方法 1. `.env` ファイルを使う

Shopify CLI では、`.env` ファイルに書いた環境変数が、TypeScript/JavaScript の `process.env.<変数名>` に埋め込まれます。

例えば、以下の内容をビルドすると、

```ts:extensions/ext1/src/index.ts
console.log(process.env.FOO);
```

```sh:.env
FOO=hogehoge
```

このように埋め込まれます。

```js:extensions/ext1/dist/ext1.js
console.log("hogehoge");
```

他のフレームワークでもおなじみのパターンですね。

### 方法 2. Shopify CLI 実行時にコマンドラインで指定する

また、`.env` ファイルに書いた環境変数だけでなく、Shopify CLI 実行時の環境変数も埋め込まれます。

```bash
$ FOO=hogehoge shopify app dev
$ FOO=hogehoge shopify app build
$ FOO=hogehoge shopify app deploy
```

### 補足

:::details Shopify CLI 3.63.2 の実装を雑にたどってみる

Shopify CLI は ESBuild を使って TypeScript/JavaScript のビルドを行っています。

https://github.com/Shopify/cli/blob/3.63.2/packages/app/src/cli/services/extensions/bundle.ts#L54-L65

ESBuild のオプションは下の `getESBuildOptions()` で組み立てられていますが、ここに `process.env.${key}` を [`define`](https://esbuild.github.io/api/#define) する設定がありますね。Webpack の `DefinePlugin` と同じようなものです。

https://github.com/Shopify/cli/blob/3.63.2/packages/app/src/cli/services/extensions/bundle.ts#L123-L149

引数の `processEnv`（`process.env`）は、Shopify CLI 実行時の環境変数です（方法 2）。また、`options.env` の出どころは以下で、これは `.env` から読み込んだ内容です（方法 1）。

https://github.com/Shopify/cli/blob/3.63.2/packages/app/src/cli/services/build/extension.ts#L88
:::

:::message
ただし、theme app extension の `assets/` に入れた JavaScript ファイルでは、環境変数の埋め込みは一切行われません。この JavaScript は Shopify CLI でビルドされるわけではなく、単にコピーされるだけです。
:::

## `shopify.extension.toml` に環境変数を埋め込む

app extension の設定ファイル `shopify.extension.toml` は拡張子の通り TOML 形式ですが、TOML には環境変数を展開するような機能はありません。また、Shopify CLI もそのような機能は持っていません。というわけで、自前で埋め込んでやる必要があります。

例として、[Shopify Flow アクション](https://shopify.dev/docs/apps/build/flow/actions/reference#action-extension-fields)の `runtime_url` を

- 本番環境では `https://production.example.com/path/to/runtime`
- ステージング環境では `https://staging.example.com/path/to/runtime`

と切り替えたいようなケースを考えます。

やり方はいろいろありそうですが、まず、テンプレートとなる設定ファイル `shopify.extension.template.toml` を用意します。

```toml:extensions/ext2/shopify.extension.template.toml
runtime_url = "https://${STAGE}.example.com/path/to/runtime"
```

このテンプレートに環境変数を埋め込んで、`shopify.extension.toml` を生成するようなスクリプトを用意しましょう。

```sh:generate-toml.sh
envsubst < extensions/ext2/shopify.extension.template.toml
  > extensions/ext2/shopify.extension.toml

# 以下でも同じ
# sed s/\${STAGE}/${STAGE}/ extensions/ext2/shopify.extension.template.toml
#  > extensions/ext2/shopify.extension.toml
```

あとはこのスクリプトを `shopify app dev|build|deploy` の前に実行してやれば OK ですね。

```console
$ STAGE=production ./generate-toml.sh && shopify app dev
```

```toml:extensions/ext2/shopify.extension.toml
runtime_url = "https://production.example.com/path/to/runtime"
```

## Liquid ファイルに環境変数を埋め込む

theme app extension には Liquid ファイルが必要です。が、Liquid の仕様上、やはり環境変数にアクセスする手段がありません。残念でした。

これも上の `shopify.extension.toml` と同じく、自前で環境変数を埋め込む必要があります。

## おわりに

Shopify アプリの開発をしていれば、環境変数を埋め込みたい場面が普通に出てくると思うんですが、Shopify の公式ドキュメントにはなぜか明記されていないんですよね。。

この記事が役に立てば幸いです。
