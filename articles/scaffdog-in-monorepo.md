---
title: "scaffdog をモノレポ環境で使う"
emoji: "🐶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [scaffdog, monorepo, react]
published: true
publication_name: "socialplus"
---

scaffolding ツール [scaffdog](https://scaff.dog/) をモノレポ環境で使おうとしたら、相対パスの解釈がちょっとややこしかったので、備忘録を兼ねて記事にしておきます。

なお、この記事では scaffdog の基本的な使い方については説明していません。

## 環境

- [scaffdog](https://scaff.dog/) v4.0.0
- [scaffdog 公式の VS Code 拡張](https://marketplace.visualstudio.com/items?itemName=scaffdog.scaffdog-vscode) v0.1.0

ちなみに弊チームではモノレポの管理に Yarn と Turborepo を使っていますが、何を使っているかは scaffdog には無関係なはずです。

## やりたいこと

このモノレポには、例として Next.js 製のアプリ `app1`, `app2` と、両アプリから使う共通コンポーネント `components` があるとします。

```
/
├── apps/
│   ├── app1/
│   └── app2/
└── packages/
    └── components/
```

アプリ `app1` や `app2` に新しいページ `foo` を追加する場合、以下の 3 ファイルを scaffold してくれると楽です。

- `foo/page.tsx`：ページの実装
- `foo/page.stories.tsx`：Storybook の story
- `foo/page.spec.tsx`：React Testing Library によるテスト

ただし、`app1` と `app2` ではそれぞれアプリ固有の事情があり、`app1` 向けのテンプレートと `app2` 向けのテンプレートを別々に用意する必要があるとします。ちなみに、scaffdog では「テンプレートが書かれた Markdown ファイル」のことを**ドキュメント**と呼んでいるので、以下はドキュメントという語を使います。

同様に、共通コンポーネントのパッケージ `components` にも、新しいコンポーネント（`Bar` とします）を scaffold するためのドキュメントを用意したいです。

- `Bar/index.tsx`：コンポーネントの実装
- `Bar/index.stories.tsx`：Storybook の story
- `Bar/index.spec.tsx`：React Testing Library によるテスト

また、[scaffdog 公式の VS Code 拡張](https://marketplace.visualstudio.com/items?itemName=scaffdog.scaffdog-vscode)が便利なので、こちらから使うことをメインのユースケースとして想定します。

## 最終的な構成

以下のような構成にすると、比較的使いやすくなりました。

- `scaffdog` 自体はリポジトリルート（モノレポの workspace root）の `devDependencies` としてインストールする
- 設定ファイルはリポジトリルートの `/.scaffdog/config.js` に配置
- 各アプリ／パッケージのドキュメントは、それぞれのアプリ／パッケージ内の `.scaffdog/` に入れる

```
/
├── package.json   👈 ここの devDependencies として scaffdog をインストール
├── .scaffdog/
│   └── config.js  👈 設定ファイル
├── apps/
│   ├── app1/
│   │   └── .scaffdog/  👈 app1 向けのドキュメント置き場
│   │       └── page.md
│   └── app2/
│       └── .scaffdog/  👈 app2 向けのドキュメント置き場
│           └── page.md
└── packages/
    └── components/
        └── .scaffdog/  👈 components 向けのドキュメント置き場
            └── component.md
```

設定ファイルの内容は以下のとおりです。各アプリ・パッケージのドキュメント置き場を `files` に指定するだけです。ここでは、**設定ファイルから見た相対パス**として解釈されます。

```js:.scaffdog/config.js
export default {
  files: ["../apps/*/.scaffdog/*", "../packages/*/.scaffdog/*"],
};
```

ドキュメントの中身は以下のとおりです。

````md:apps/app1/.scaffdog/page.md
---
name: '[app1] page'
root: 'apps/app1/src/app/'
output: '**/*'
ignore: []
questions:
  name: 'Please enter the page path.'
---

# `{{ inputs.name }}/page.tsx`

```tsx
import { usefulFunc } from '{{ relative '../src/usefulFunc' }}';

/* 略 */
```

（後略）
````

いくつか注意点があるので、順に見ていきます。

### `name`

まず、front matter で設定する `name`（ドキュメント名）は `[app1] page` や `[app2] page` のように、区別できる名前にしておく必要があります。

![](https://storage.googleapis.com/zenn-user-upload/8a473aaf1882-20240820.png)

形式は何でもいいんですが、区別しておかないと scaffdog を実行したいときに困ります。

![](https://storage.googleapis.com/zenn-user-upload/a83bbe8dda68-20240820.png)

### `root`, `ignore`

`root` や `ignore` のパスは、**「scaffdog を起動したディレクトリ」から見た相対パス**として解釈されるようです。今回はリポジトリルートに `scaffdog` をインストールし、そこで起動することにしました。VS Code 拡張から起動する場合も、自動的にそうなります。

そこで、例えば `app1` の `src/app/` を `root` としたい場合は、リポジトリルートから見た相対パス `apps/app1/src/app/` として書く必要があります。

### `relative` helper

相対インポートのパスは [`relative` helper](https://blog.wadackel.me/2019/scaffdog/#%E4%BE%BF%E5%88%A9%E3%81%AA%E3%83%93%E3%83%AB%E3%83%88%E3%82%A4%E3%83%B3%E9%96%A2%E6%95%B0%E7%BE%A4) を使うと出力できます。この `relative` helper の引数に指定するパスは、**インポートしたいファイルを、そのドキュメントから見た相対パス**です。

例えば `app1` において、ドキュメント `.scaffdog/page.md` からページのファイル `src/app/foo/page.tsx` を生成する場合を考えます。そして、`src/usefulFunc.ts` をインポートしたいとします。

```
/
├── apps/
│   ├── app1/
│   │   ├── .scaffdog/
│   │   │   └── page.md  👈 ドキュメント
│   │   └── src/
│   │       ├── app/
│   │       │   └── foo/
│   │       │       └── page.tsx  👈 生成先
│   │       └── usefulFunc.ts  👈 インポートしたいファイル
```

このとき、インポートしたいファイルの相対パスはドキュメントから見ると `../src/usefulFunc.ts` なので、これを `relative` helper の引数に渡して以下のようにインポート文を書きます。

```tsx:apps/app1/.scaffdog/page.md
import { usefulFunc } from '{{ relative '../src/usefulFunc' }}';
```

すると、生成したファイルの中では以下のようになります。

```tsx:apps/app1/src/app/foo/page.tsx
import { usefulFunc } from '../../src/usefulFunc';
```

ちなみに、import alias を使えば相対インポート自体が不要になりますが、弊チームでは採用していません。この判断は、azu さんの以下の記事を参考にしたものです。

> alias 的に使えるが、path をファイルパスのショートカットとして使うのはツール解析を難しくするので使わない。

https://gist.github.com/azu/56a0411d69e2fc333d545bfe57933d07#paths

## ボツ案

いろいろ考えてボツになった案も紹介しておきます。

### A. ドキュメントをリポジトリルートの `.scaffdog` に集約する

```
/
├── package.json   👈 ここの devDependencies として scaffdog をインストール
├── .scaffdog/
│   ├── config.js
│   ├── app1-page.md
│   ├── app2-page.md
│   └── component.md
├── apps/
│   ├── app1/
│   └── app2/
└── packages/
    └── components/
```

`relative` helper に書く相対パスが `../apps/app1/src/usefulFunc` のように長ったらしくなるのでボツにしました。また、アプリ固有のドキュメントはアプリ実装の近くに置いておきたいところですが、この構成だと離れてしまうという欠点もあります。

### A'. 独自 helper を作って A. をマシにする

ボツ案 A. の「`relative` helper に書く相対パスが長ったらしい」という点の対策として、「もっと簡単に書ける独自 helper を用意する」ことも考えました。

`app1` の相対インポートでは、`app1` のパッケージ内部のファイルしかインポートしません。そこで、この制約を利用します。ドキュメントはリポジトリルートの `.scaffdog` に集約しますが、どのアプリ／パッケージ向けなのか、`.scaffdog` 以下のディレクトリで区別できるようにしておきます。

```
/
├── package.json   👈 ここの devDependencies として scaffdog をインストール
├── .scaffdog/
│   ├── config.js
│   ├── apps/
│   │   ├── app1/
│   │   │   └── page.md
│   │   └── app2/
│   │       └── page.md
│   └── packages/
│       └── components/
│           └── component.md
├── apps/
│   ├── app1/
│   └── app2/
└── packages/
    └── components/
```

独自 helper `relativeInPackage` の実装は以下のとおりです。

```ts:.scaffdog/config.ts
import path from 'node:path';
import { Config } from '@scaffdog/types';

export default {
  files: ['**/*', '!README.md'],
  helpers: [
    {
      /**
       * import 文のパスを、パッケージのルートディレクトリを基準として書けるようにするための helper
       *
       * scaffdog 組み込みの `relative` helper では、document（Markdown ファイル）を基準として引数のパスが解釈されるが、
       * この helper では、ファイル生成先のパッケージのルートディレクトリが基準となる。
       *
       * @param ctx scaffdog のコンテキスト
       * @param pathFromPackageRoot ファイル生成先のパッケージのルートディレクトリから見た相対パス
       * @returns 生成したファイルから見た相対パス
       *
       * @example
       * // app1 向けのテンプレートに以下のように記述し、
       * import { usefulFunc } from '{{ relativeInPackage 'src/usefulFunc' }}';
       *
       * // `apps/app1/src/app/foo/page.tsx` を生成した場合、以下のような内容になる
       * import { usefulFunc } from '../../foo';
       */
      relativeInPackage: (ctx, pathFromPackageRoot: string) => {
        const packageRoot =
          // 選択した document（Markdown ファイル）のディレクトリ
          // e.g. /path/to/repo/.scaffdog/apps/app1
          (ctx.variables.get('document') as { dir: string }).dir
            // そこから `.scaffdog/` を取り除いたものが、生成先のパッケージのルートディレクトリ
            // e.g. /path/to/repo/apps/app1
            .replace('.scaffdog/', '');

        // 生成されるファイルのディレクトリ
        // e.g. /path/to/repo/apps/app1/src/app/foo
        // dir だと VS Code 拡張から実行した場合は相対パス、CLI では絶対パスになるので、abs の方から取得する
        const outputDir = path.dirname(
          (ctx.variables.get('output') as { abs: string }).abs,
        );

        return path.relative(
          outputDir,
          path.join(packageRoot, pathFromPackageRoot),
        );
      },
    },
  ],
} satisfies Config;
```

こうすると、以下のような相対インポートが

```tsx:.scaffdog/apps/app1/page.md
import { usefulFunc } from '{{ relative '../../../app/app1/src/usefulFunc' }}';
```

独自 helper `relativeInPackage` によって以下のように簡潔に書けます。

```tsx:.scaffdog/apps/app1/page.md
import { usefulFunc } from '{{ relativeInPackage 'src/usefulFunc' }}';
```

独自 helper の仕組みを考えたり書いたりとても楽しかったんですが、上に挙げた「最終的な構成」にすればシンプルかつ十分なことが分かったため、あえなくボツになりました。

### B. アプリ／パッケージごとに scaffdog を入れる

```
/
├── apps/
│   ├── app1/
│   │   ├── package.json   👈 devDependencies として scaffdog をインストール
│   │   └── .scaffdog/
│   │       ├── config.js
│   │       └── page.md
│   └── app2/
│       ├── package.json   👈 devDependencies として scaffdog をインストール
│       └── .scaffdog/
│           ├── config.js
│           └── page.md
└── packages/
    └── components/
        ├── package.json   👈 devDependencies として scaffdog をインストール
        └── .scaffdog/
            ├── config.js
            └── component.md
```

一番シンプルな方法です。

ただし、モノレポ全体を VS Code で開いていると、VS Code 拡張がうまく動いてくれません。拡張の設定で `.scaffdog` のパスを指定することができ、カンマ区切りで複数指定することもできます。

![](https://storage.googleapis.com/zenn-user-upload/90a75324f8b3-20240820.png)

しかしこうした場合、一番最初見つかった `.scaffdog` が参照され、それ以外は無視されるようです。つまり、上の設定では `app1` のドキュメントは VS Code 拡張から利用できますが、`app2` や `components` のドキュメントは利用できません。

### C. ドキュメント内でアプリごとに条件分岐させる

頑張ればできそうではありますが、メンテナンスが大変そうなのでやめました。チームの方針としても、現状は「あまり頭の良いことはせずに、シンプルに scaffdog を使う」ことにしています。

## まとめ

ということで、モノレポに scaffdog を入れる際の構成例を紹介しました。相対パスの扱いがちょっとややこしいですが、一度設定してしまえば後は困ることもなく、どんどん使っていけそうです。
