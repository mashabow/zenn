---
title: "AI に codemod を書かせて大規模リファクタリングに立ち向かう"
emoji: "📜"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["codemod", "リファクタリング", "生成ai"]
published: true
publication_name: "socialplus"
---

リファクタリングをしていて、ガッと数十ファイルにわたって一括で書き換えたいような場面、みなさんもありますよね？ 場合によっては数百かもしれません。

（「そんなに広範囲に影響する時点で設計が悪いのでは？」という指摘はあるかもしれませんが、この記事では横に置いておきます。）

この記事は、「**AI に codemod を書かせて、それを実行すると書き換えが楽**ですよ」というお話です。codemod のことを全く知らなくても、AI に書いてもらえば今日から使っていけるはずです。やることは簡単で、要は以下の手順を回すだけです。

1. 「これをこう書き換える codemod を作って」と AI にお願いする
2. codemod を実行して書き換える
3. 書き換え結果を確認
   - 満足できなかったら、codemod を調整して 2 に戻る
4. 書き換え完了

なお、TypeScript を例に説明をしていますが、言語を問わずに応用できるはずです。

## 前置き

急いでいる人はすっとばして、[AI に codemod を書いてもらおう](#ai-%E3%81%AB-codemod-%E3%82%92%E6%9B%B8%E3%81%84%E3%81%A6%E3%82%82%E3%82%89%E3%81%8A%E3%81%86)に進んでください。

### そもそも codemod とは？

codemod は、**コードの書き換えを一括で行うツール**の総称です。TypeScript/JavaScript では、[jscodeshift](https://github.com/facebook/jscodeshift) や [ts-morph](https://ts-morph.com/) がよく使われています。また、書き換え用のスクリプトを指して codemod と呼ぶこともあります。もともとは Python 向けの書き換えツール [codemod](https://github.com/facebookarchive/codemod) の名前でしたが、今では一般名詞化しています。

「書き換え」でまずイメージするのは「正規表現を使った文字列置換」だと思いますが、codemod では **AST（抽象構文木）を操作して書き換えを行います**。そのため、文法構造を壊さずに柔軟に書き換えができます。普段「HTML は文字列そのままじゃなくて DOM を操作して書き換えろ」「URL も文字列ではなく、[`URL` オブジェクト](https://developer.mozilla.org/ja/docs/Web/API/URL) にしてから操作した方が安全」みたいな話を聞きますよね。AST にしてから操作するのはあれと同じ発想です。

最近では、ライブラリのアップデートを支援するために、ライブラリ側で codemod を提供している例もよく見かけます。使ったことある人もいるんじゃないでしょうか。以下は Next.js の例です。

- [Upgrading: Codemods | Next.js](https://nextjs.org/docs/app/guides/upgrading/codemods)

誰かが作った codemod を使うだけであれば簡単なんですが、自分で書こうとすると AST の知識が必要になってきます。AST を操作するための API も知っておく必要があり、ちょっとハードルが高いですよね。

https://zenn.dev/did0es/articles/17a503c6398c27

https://speakerdeck.com/ypresto/ts-morph-codemod-and-type

ということで、AI さんに書いてもらうことにしましょう。

### 実は AI は単純な書き換えが苦手？

半年ほど前、AI エージェントを業務で使い始めたときのことです。当初は「AI に書き換えをお願いすれば、大規模なリファクタリングも楽勝じゃん？」と思っていたんですが、やらせてみると彼らは案外苦手です。期待とのギャップが結構あったのを覚えています。

たとえば、

```ts
import { Foo } from 'awesome-package';
```

を一括で

```ts
import { Foo } from './foo';
```

に書き換えるようなケースを考えます。これを AI さんに直接頼むと、ソースコードを単純に grep して、文字列置換で解決しようとしたりします。すると、

```ts
import {
  Foo,
  Bar,
  Baz,
} from 'awesome-package';
```

のようなインポート文の `Foo` の書き換えが漏れてしまったり、書き換え後のコードが文法エラーになっていたり……。また、書き換え後の相対パスの階層が食い違っていることもあります。

AI エージェントなら壊れたことに気づいて、直そうとはしてくれるんですが、「ファイルを読み込んで、直してみて、エラーが出ていないかチェックして……」ということを繰り返したりします。その都度トークンを消費しますし、スピードも思うように出ません。

## AI に codemod を書いてもらおう

やることは単純で、素直にお願いするだけです。

> ○○○を×××に一括で書き換えたいと思います。この書き換えを行う codemod を作成してください。

以下はその例です。

> ```ts
> import { Foo } from 'awesome-package';
> ```
>
> を
>
> ```ts
> import { Foo } from './foo';
> ```
>
> に一括で書き換えたいと思います。この書き換えを行う codemod を作成してください。
>
> 書き換え後の `./foo` の箇所は、実際には `src/foo.ts` を指す相対パスが入ります。

こうお願いしたら、次のような codemod が作成されました。コメントも原文ママです。

```ts
import { FileInfo, API } from 'jscodeshift';
import * as path from 'path';

export default function transform(
  file: FileInfo,
  api: API,
) {
  const j = api.jscodeshift;
  const root = j(file.source);

  // awesome-package からの import を検出
  root
    .find(j.ImportDeclaration, {
      source: {
        value: 'awesome-package',
      },
    })
    .forEach((pathNode) => {
      const specifiers = pathNode.node.specifiers || [];
      const fooSpecifier = specifiers.find(
        (s) => s.type === 'ImportSpecifier' && s.imported.name === 'Foo'
      );
      if (!fooSpecifier) {
        return;
      }

      // src/foo.ts への相対パスを計算
      const sourceDir = path.dirname(file.path);
      const relativePath = path
        .relative(sourceDir, 'src/foo.ts')
        .replace(/\.ts$/, ''); // 拡張子を除去
      const relativeImportPath = relativePath.startsWith('.')
        ? relativePath
        : `./${relativePath}`; // 相対パスが . で始まっていない場合は ./ を追加

      // Foo のみを新しい import 文として追加
      const newImport = j.importDeclaration(
        [j.importSpecifier(j.identifier('Foo'))],
        j.stringLiteral(relativeImportPath)
      );
      pathNode.insertBefore(newImport);

      // 元の import から Foo を削除
      const remainingSpecifiers = specifiers.filter(
        (s) => s.type !== 'ImportSpecifier' || s.imported.name !== 'Foo'
      );
      if (remainingSpecifiers.length === 0) {
        // 他の import がない場合は import 文自体を削除
        pathNode.prune();
      } else {
        // 他の import がある場合は Foo のみを削除
        pathNode.node.specifiers = remainingSpecifiers;
      }
    });

  return root.toSource();
}
```

これを一から手で書くのはなかなか大変そうですね。

ちなみに、単にお願いするだけだと、AI エージェントによっては codemod 中のリントエラーや型エラーを律儀に直そうとしてくるかもしれません。そこはこだわるところではないので、

> codemod は使い捨てのスクリプトなので、リントエラーや型エラーは無視して構いません。

と指示するといいかと思います。

:::message

codemod を書いてもらったら、この時点で一旦 codemod のファイルをコミットしておくのが個人的にはおすすめです。この後の調整がやりやすくなります。また、codemod は使い捨てのスクリプトなので、作業が終わったら消すことにはなりますが、一旦コミットしておけば「この書き換えが妥当か」のレビューがしやすくなります。

:::

## 書いてもらった codemod を動かしてみる

codemod ができたら、さっそく試しに動かしてみましょう。

> いま作った codemod を実行してください

のように頼んでもいいですし、自分で実行してもかまいません。実行方法がわからなければ、AI さんに聞けば教えてくれます。

動かした後（書き換え後）のコードを確認してみると、「こういうときはこう書き換えてほしかった」とか「このケースに対応できてなかった」というような改善点が見つかったりします。以下のような選択肢で適宜対応します。

- AI に改善点を伝えて、codemod を調整してもらう
- 自分で codemod をいじって調整する
- レアなケースは無理をせずにあきらめて、個別に（AI or 人）が書き換える

調整してもらったらまた実行してみましょう。このサイクルの繰り返しです。

納得のいく書き換えができたら、最後に codemod のスクリプトを削除して書き換え完了です。おつかれさまでした。

## まとめ

ということで、AI に各ファイルを直接書き換えてもらうのではなく、codemod を書いてもらってそれを実行する方法を紹介しました。メリットとしては以下のとおりでしょうか。

- 書き換えが速い
- 書き換えに再現性がある
- 書き換えのロジックがコードとして表現されている
- codemod の書き方を覚えなくても使える

特に「再現性がある」「コードとして表現されている」については、リファクタリングをする側だけでなく、レビュワーにとっても地味に助かるところです。

なお今回は、各ステップで人間が介在する形のフローを紹介しました。まずはこれを試してみると、codemod 活用のイメージが掴めると思います。codemod に慣れてきたら、もっと AI 任せにしてしまっても良いかもしれません。

codemod の力を借りて、大規模リファクタリングに立ち向かっていきましょう 💪
