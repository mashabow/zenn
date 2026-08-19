---
title: "ログの秘匿情報マスク漏れを OpenAPI で防いだ話"
emoji: "⬛️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["openapi", "logging", "logrocket"]
published: true
publication_name: "socialplus"
---

HTTP リクエスト／レスポンスのログを吐くとき、その中に秘匿情報が含まれていたら、きっちりマスクしないとダメですよね。

```ts
// マスク前
[
  { id: '123', name: '佐藤ほげ丸', role: 'admin' },
  { id: '456', name: '鈴木ふが美', role: 'member' },
  { id: '789', name: '高橋ぴよ助', role: 'member' },
]

// マスク後
[
  { id: '123', name: '[MASKED]', role: 'admin' },
  { id: '456', name: '[MASKED]', role: 'member' },
  { id: '789', name: '[MASKED]', role: 'member' },
]
```

ここでいう秘匿情報とは、以下のようなものを指すことにします。

- 氏名や住所のような個人情報
- パスワードやシークレットのような認証情報

しかし、一度「きっちりマスクできた！」というだけでは終わりではありません。今後は**それを保守して、マスク漏れのない状態をきっちり維持する必要**があります。

わたしたちのプロダクトでも、実は昔はマスク漏れ（未遂）が多発して、頭を悩ませていました。そしてあるとき「これじゃまずい」と思い、OpenAPI を使ってマスクの方法を工夫してみました。この記事では、その内容を紹介していきます。

ちなみにこの方法を導入して3年ほど経ちますが、今ではマスク漏れはほとんどありません。

:::message
簡単のため、以下ではリクエスト／レスポンスのボディに絞って議論します。ヘッダーなどは無視します。
:::

## マスク漏れが起きやすかったころの状況

その昔、マスク漏れが起きやすかったころの方法をまず説明しておきます。先を急ぐ人は読み飛ばしてください。

そもそも、どうしてマスク漏れが起きやすかったんでしょうか？

### 当時の方法

当時は、「どのリクエスト／レスポンスボディに含まれる、どのプロパティをマスクするか？」を、次のような配列で持っていました。（以下、マスク定義と呼びます）

```ts
const masks: readonly {
  /** エンドポイントのパスのパターン */
  readonly path: RegExp;
  /** マスク対象のプロパティ名 */
  readonly keys: readonly string[];
}[] = [
  {
    path: /\/hoge\/fuga/,
    keys: ['name', 'address'],
  },
  {
    path: /\/foo\/[^/]+\/bar/, // パスパラメータを含むケース
    keys: ['secret'],
  },
  ...
];
```

これを手で書いて定義していたわけですが、以下のような問題がありました。

### この方法の問題点

まず、エンドポイントの指定（`path`）が正規表現ベースなので、書きにくくて読みにくいのです。上の例では簡単なものしか挙げていませんが、実際には複雑なものもありました。ちゃんと書いたつもりでも定義ミスが起こり得ますし、レビューも地味にすり抜けそうです。

また細かいところでは、「マスク対象のプロパティがプロパティ名（`keys`）だけで指定されており、階層が考慮されていない」という問題もあります。複数のオブジェクトを寄せ集めたリクエスト／レスポンスボディでは、`name` のようなプロパティが複数個所で出てくるかもしれません。雑な例だとこんな感じです。

```json
{
  "user": {
    "id": "1234",
    "name": "ななしのごんべえ", // ここをマスクしたいのに
    "likes": [
      {
        "id": "abcd",
        "name": "かっこいいTシャツ", // こっちまでマスクされてしまう
      }
    ],
  },
}
```

同名のプロパティが巻き添え食らって余計にマスクされてしまいます。また、今後保守していくにあたっても、マスク定義を見ただけでは「どこの `name` をマスクしたいんだっけ？」というのが分からず、困りそうです。

さらに、それ以前の問題として、新しいエンドポイントを生やしたときに**単純にマスク定義の追加を忘れていた**こともありました。同様に、**API 側を仕様変更した際に、マスク定義側の追従を忘れていた**というケースもあります。例えば、リネームやプロパティの追加などですね。

問題点をまとめると、以下のとおりです。

- API 仕様書とマスク定義が別々で管理されているため、追加漏れや追従漏れが起きやすい
- 手でマスク定義を書いているため、人為的なミスが入り込みやすい
- マスク定義を見ただけではマスク対象の中身がはっきりわからないため、保守がしづらい

これらの要因が重なって、マスク漏れが起きやすい状況になっていました。

## OpenAPI の拡張を使った解決策

ということで、これらの要因をつぶすことができれば、マスク漏れの問題が解決しそうです。そう考えて以下の方針を立てました。

1. 「秘匿情報か否か」のフラグを API 仕様書と一緒に管理することで、対応関係を明確にし、追加漏れや追従忘れを防ぐ
2. そこからマスク対象のリストを自動生成し、人為的なミスが入り込む余地を減らす

これを実現していきます。

### 設計

設計は以下のとおりです。**API の仕様は OpenAPI で管理していたので、ここに「秘匿情報か否か」のフラグを追加し、一元管理する**ことにしました。

1. OpenAPI ドキュメント上のリクエスト／レスポンスボディにおいて、秘匿情報のプロパティに `x-socialplus-sensitive: true` という拡張フィールドを追加する
2. OpenAPI ドキュメントを読みこんでマスク対象の一覧を抽出し、マスク定義の JSON ファイルを生成する
3. アプリはマスク定義の JSON ファイルをインポートし、そのデータをもとにリクエスト／レスポンスボディのマスク処理を行う

順に説明していきます。

> 1. OpenAPI ドキュメント上のリクエスト／レスポンスボディにおいて、秘匿情報のプロパティに `x-socialplus-sensitive: true` という拡張フィールドを追加する

このようなイメージです。

```yaml
paths:
  /users/me:
    get:
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  name:
                    type: string
                    description: ユーザーの氏名
                    x-socialplus-sensitive: true  # この行を追加
                  ...
```

OpenAPI では、**specification extension** という標準的な拡張方法が定められています（[ガイド](https://swagger.io/docs/specification/openapi-extensions/)・[仕様](https://spec.openapis.org/oas/v3.2.0.html#specification-extensions)）。`x-` で始まる名前のフィールドがそれで、利用者が自由に定義して使うことができます。今回は、自社で勝手に決めたものだということが明確になるように、社名 `socialplus` を入れて `x-socialplus-sensitive` という名前にしています。


別の手段として、[`format`](https://spec.openapis.org/oas/v3.2.0.html#data-type-format) フィールドにカスタムの値を使って `format: socialplus-sensitive` のようにする手も考えられますが、`format` を普通の用途で使いたくなったときに困るのでやめました。

ちなみに、「秘匿情報を示すための専用のフィールドを標準化してほしい」という issue もありますが、あまり議論は進んでいないようです。

https://github.com/OAI/OpenAPI-Specification/issues/2190

> 2. OpenAPI ドキュメントを読みこんでマスク対象の一覧を抽出し、マスク定義の JSON ファイルを生成する

以下のようなファイルを生成します。

```json
[
  {
    "path": "/users/me",
    "method": "get",
    "masks": {
      "requestBody": [],
      "responseBody": ["$.name"]
    }
  },
  {
    "path": "/users/me",
    "method": "patch",
    "masks": {
      "requestBody": ["$.name"],
      "responseBody": ["$.name"]
    }
  },
  {
    "path": "/users",
    "method": "get",
    "masks": {
      "requestBody": [],
      "responseBody": ["$[*].name"]
    }
  },
  {
    "path": "/users/{userId}",
    "method": "get",
    "masks": {
      "requestBody": [],
      "responseBody": ["$.name"]
    }
  },
  ...
]
```

エンドポイントのパスの指定には、OpenAPI の形式 `/foo/{bar}` をそのまま使いました。正規表現だと表現力が過剰です。

また、プロパティの指定には、[JSONPath](https://www.rfc-editor.org/info/rfc9535/) 形式を採用しました。当時は標準化されてなかったんですが、いま調べてみたら2024年に RFC になってたんですね。似たようなものに [JSON Pointer](https://www.rfc-editor.org/info/rfc6901/) がありますが、JSON Pointer だと「配列の全ての要素の中の `name`」のような指定ができず、実用上困ります。JSONPath であれば `$[*].name` のようにワイルドカードで表現できます。

> 3. アプリはマスク定義の JSON ファイルをインポートし、そのデータをもとにリクエスト／レスポンスボディのマスク処理を行う

中間ファイル（マスク定義の JSON ファイル）を作らずに、2 と 3 をまとめてしまう方法も考えられますが、以下の理由で 2 と 3 を分離しています。

- フロントエンドにおいては、バンドルに OpenAPI ドキュメント全体を入れることは避けたい。必要な情報だけを抽出した、マスク定義の JSON のみに絞りたい。
- JSON であれば手書きもできるため、徐々に移行できる。当時は OpenAPI 化されていないエンドポイントがまだ残っていたため、その部分については手書きで補っていた。

### 実装

設計が決まればあとは実装するだけです。実装したのは3年前で、当時は ChatGPT の助けを借りつつ自分で実装していましたが、今なら AI がほとんど作ってくれそうですね。

> 2. OpenAPI ドキュメントを読みこんでマスク対象の一覧を抽出し、マスク定義の JSON ファイルを生成する

:::details ソースコード

```ts
import fs from 'fs';

import Oas, { Operation } from 'oas';
import OASNormalize from 'oas-normalize';
import { OpenAPIV3, OpenAPIV3_1 } from 'openapi-types';

export interface LogMask {
  /** エンドポイントのパス。OpenAPI の path の形式と同様 */
  readonly path: string;
  /** HTTP メソッド。OpenAPI の method の形式と同様 */
  readonly method: string;
  readonly masks: {
    /** リクエストボディ中のマスクすべきプロパティの一覧。配列の各要素は JSONPath 形式で表現したプロパティ名 */
    readonly requestBody: readonly string[];
    /** レスポンスボディ中のマスクすべきプロパティの一覧。配列の各要素は JSONPath 形式で表現したプロパティ名 */
    readonly responseBody: readonly string[];
  };
}

export type LogMasks = readonly LogMask[];

/**
 * オブジェクトのスキーマを再帰的に探索し、マスク対象のプロパティをすべて列挙する
 *
 * @returns マスク対象のプロパティのパスの配列。JSONPath 形式。
 */
const collectMasks = (
  schema:
    | OpenAPIV3.SchemaObject
    | OpenAPIV3.ReferenceObject
    | OpenAPIV3_1.SchemaObject
    | OpenAPIV3_1.ReferenceObject
    | undefined,
  currentPath = '$',
): readonly string[] => {
  if (!schema) return [];
  if ('$ref' in schema) throw new Error('`schema` must be dereferenced.');

  if ('x-socialplus-sensitive' in schema && schema['x-socialplus-sensitive']) {
    return [currentPath];
  }

  // type: object
  if (schema.properties) {
    return Object.entries(schema.properties).flatMap(([key, value]) =>
      collectMasks(value, `${currentPath}.${key}`),
    );
  }

  // type: array
  if ('items' in schema) {
    return collectMasks(schema.items, `${currentPath}[*]`);
  }

  return [
    ...(schema.allOf ?? []),
    ...(schema.oneOf ?? []),
    ...(schema.anyOf ?? []),
  ].flatMap((ofItem) => collectMasks(ofItem, currentPath));
};

const collectRequestBodyMasks = (operation: Operation): readonly string[] => {
  const { requestBody } = operation.schema;
  if (!requestBody) return [];

  if ('$ref' in requestBody) throw new Error('`schema` must be dereferenced.');
  return collectMasks(requestBody?.content?.['application/json']?.schema);
};

const collectResponseBodyMasks = (operation: Operation): readonly string[] =>
  // レスポンスはステータスコードごとに定義されているが、区別せずに一緒くたに扱う
  Object.values(operation.schema.responses ?? {}).flatMap(
    (response: OpenAPIV3.ResponseObject | OpenAPIV3_1.ResponseObject) =>
      collectMasks(response.content?.['application/json']?.schema),
  );

/**
 * OpenAPI ドキュメントを読みこんでマスク対象のプロパティを抽出し、JSON ファイルに出力する
 *
 * @param inputFile OpenAPI ドキュメントの YAML ファイルのパス
 * @param outputFile マスク対象の JSON ファイルのパス
 */
export const generate = async (
  inputFile: string,
  outputFile: string,
): Promise<void> => {
  const document = await new OASNormalize(inputFile, {
    enablePaths: true,
  }).deref();
  const oas = new Oas(document as any);

  const operations: readonly Operation[] = Object.values(
    oas.getPaths(),
  ).flatMap(Object.values);

  const logMasks: LogMasks = operations
    .map((operation) => ({
      path: operation.path,
      method: operation.method,
      masks: {
        requestBody: collectRequestBodyMasks(operation),
        responseBody: collectResponseBodyMasks(operation),
      },
    }))
    .filter(
      ({ masks }) =>
        masks.requestBody.length > 0 || masks.responseBody.length > 0,
    );

  fs.writeFileSync(outputFile, JSON.stringify(logMasks, null, 2), 'utf-8');
};
```

:::

OpenAPI ドキュメントの取り扱いは、npm パッケージ [`oas`](https://www.npmjs.com/package/oas) を使うと便利です。ただし、YAML 形式の OpenAPI ドキュメントを読み込む際には、先に [`oas-normalize`](https://www.npmjs.com/package/oas-normalize) を使って変換する必要があります。

再帰的にプロパティをたどって `x-socialplus-sensitive` を探し、JSONPath を組み立てれば OK です。

> 3. アプリはマスク定義の JSON ファイルをインポートし、そのデータをもとにリクエスト／レスポンスボディのマスク処理を行う

:::details ソースコード

```ts
import { JSONPath } from 'jsonpath-plus';

// 2 で OpenAPI から生成した、マスク定義の JSON
import LOG_MASKS from './logMasks.json';

const MASKED_VALUE = '[MASKED]';
const BASE_URL = 'https://example.com/api/v1';

/**
 * 適用すべき LogMask を選択する
 *
 * リクエスト／レスポンスのエンドポイントと LogMask + baseUrl を比較して、以下の3つが一致しているものを選択する
 *
 * - a. HTTP メソッド
 * - b. URL の origin
 * - c. URL の pathname
 *
 * @param endpoint リクエスト／レスポンスのエンドポイント
 * @returns 適用すべき LogMask。見つからなければ undefined を返す
 */
const selectLogMask = (
  endpoint: { url: string; method: string },
): LogMask | undefined => {
  const url = new URL(endpoint.url);

  // b. URL の origin を比較
  if (url.origin !== new URL(BASE_URL).origin) return undefined;

  const matchedLogMasks = LOG_MASKS
    // a. HTTP メソッドを比較
    .filter(({ method }) => method === endpoint.method)
    // c. URL の pathname を比較
    .filter(({ path }) => {
      // path と BASE_URL から絶対パス pathname を求める
      // e.g. BASE_URL が `https://example.com/api/v1`、path が `/foo/{param}`, pathname は `/api/v1/foo/{param}`
      const pathname = decodeURIComponent(
        new URL(`${BASE_URL}${path}`).pathname,
      );
      // パスパラメータ部分は OpenAPI では `{param}` で表現されているので、完全一致の正規表現に変換してマッチするか判定する
      const pattern = `^${pathname}$`.replaceAll(/{.+?}/g, '[^/]+');
      return new RegExp(pattern).test(url.pathname);
    });

  // パスパラメータの個数が一番少ない LogMask を採用する
  // OpenAPI の仕様上、より具体的なパス（パスパラメータの個数が少ないパス）ほど優先度が高いので
  // https://spec.openapis.org/oas/v3.1.0#path-templating-matching
  return matchedLogMasks
    .toSorted((a, b) => a.path.split('{').length - b.path.split('{').length)
    .at(0);
};

/**
 * ボディの中の指定したプロパティをマスクする
 *
 * @param body リクエストボディまたはレスポンスボディ
 * @param maskPaths マスクすべきプロパティの一覧。配列の各要素は JSONPath 形式で表現したプロパティ名
 * @returns マスク後のボディ
 */
const maskBody = (body: {}, maskPaths: readonly string[]): {} => {
  // root object をマスクする場合は、マスクを返す
  if (maskPaths.includes('$')) return MASKED_VALUE;

  maskPaths
    .flatMap((maskPath) =>
      JSONPath({
        json: body,
        path: maskPath,
        resultType: 'all',
      }),
    )
    .forEach(({ parent, parentProperty }) => {
      parent[parentProperty] = MASKED_VALUE;
    });

  return body;
};
```

```ts:レスポンスボディをマスクする例
const endpoint = {
  url: 'https://example.com/api/v1/users',
  method: 'get',
}
const body = [
  { id: '123', name: '佐藤ほげ丸', role: 'admin' },
  { id: '456', name: '鈴木ふが美', role: 'member' },
  { id: '789', name: '高橋ぴよ助', role: 'member' },
];

const logMask = selectLogMask(endpoint);
if (!logMask) return body;
return maskBody(body, logMask.masks.responseBody);
// 以下のようにマスクされる
// [
//   { id: '123', name: '[MASKED]', role: 'admin' },
//   { id: '456', name: '[MASKED]', role: 'member' },
//   { id: '789', name: '[MASKED]', role: 'member' },
// ]
```

:::

まず、マスク定義の JSON（全エンドポイントのマスク対象が列挙されている配列）から、エンドポイントにマッチする項目を探します。上のコードの `selectLogMask()` の部分にあたります。パスパラメータの扱いにちょっと注意が必要です。

次に、その結果をもとに、実際のリクエスト／レスポンスボディの中身をマスクします。`maskBody()` の部分です。マスクすべきプロパティは JSONPath で示されているので、npm パッケージ [`jsonpath-plus`](https://www.npmjs.com/package/jsonpath-plus) を使って該当箇所をマスクしていけば OK です。

これでマスク処理ができました。

### 利用

あとは、ログ出力時にこのマスク処理を呼び出せば完了です。

ちなみに、ここで紹介したマスクの仕組みは、もともとセッションリプレイサービス [LogRocket](https://logrocket.com/) のために用意したものです。LogRocket は、サイトを訪問したユーザーの操作を記録してくれるサービスで、操作だけでなく通信内容も記録します。このログは不具合調査などで役立つのですが、秘匿情報は LogRocket には記録したくありません。そこで、上記のマスク処理を LogRocket の [`requestSanitizer` と `responseSanitizer`](https://docs.logrocket.com/reference/network) に組み込みました。

![](https://static.zenn.studio/user-upload/f3db0a9bd33e-20260813.png =400x)
*LogRocket 上における、マスクされたログの例*

もちろん LogRocket に限らず、他のログ出力でも使えます。

## まとめ

「秘匿情報か否か」を OpenAPI で一元管理して、マスク漏れを防ぐ仕組みを紹介しました。

とはいえ、いくら一元管理したところで、そもそも `x-socialplus-sensitive` をつけ忘れたら元も子もないのは事実です。ただ、最近では AI によるレビューで「`x-socialplus-sensitive` をつけ忘れてます」と指摘してくれますし、リスクはかなり減りました。

また、OpenAPI 上で明示するようになったことをきっかけとして、「何をどこまでマスク対象とすべきか」という基準の明確化が進んだのも、チームに対する副次的な効果としてありました。

これで安心して眠れそうです ☺️
