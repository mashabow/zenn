---
title: 'Google タグマネージャーでは type="module" な script 要素は実行されない'
emoji: "🫥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['gtm', 'javascript', 'dom', 'esm']
published: true
publication_name: "socialplus"
---

Google タグマネージャー（以下、GTM）でちょっとハマったので、記事にしておこうかと思います。

## `type="module"` な `script` 要素は実行されない

GTM の「カスタム HTML」で

```html
<script type="module" src="/foo.js"></script>
```

のように設定したとします。

![](https://storage.googleapis.com/zenn-user-upload/4ca635e41b9c-20251105.png)

この場合、`/foo.js` のスクリプトが実行されるかと思いきや、**実際には実行されません**。ちなみに `type="module"` をつけずに

```html
<script src="/foo.js"></script>
```

とすれば、期待どおり普通にスクリプトが実行されます。

また、インラインのスクリプトも同様で、`type="module"` がついていると実行されません。

まとめると以下のようになります。**`type="module"` なスクリプトは実行されません**。

```html
<!-- 実行されない -->
<script type="module" src="/foo.js"></script>
<script type="module">console.log('Hi!')</script>

<!-- 実行される -->
<script src="/foo.js"></script>
<script>console.log('Hi!')</script>
```

GTM 経由でサードパーティースクリプトを使うユーザーにとっても、逆にサードパーティースクリプトを開発・提供する側にとっても、これはちょっと気を付ける必要がありそうですね。

では、どうしてこうなるんでしょう？「GTM が `type="module"` をサポートしていないから」と言ってしまえばそれまでなんですが、内部的にはどうなっているのか、気になったのでちょっと見てみました。

## 実行されないけど、`script` 要素は DOM に追加されている

実は開発者ツールで見てみると、`script` 要素はちゃんと DOM に追加されているように見えます。例えば、「カスタム HTML」に以下の内容を設定すると、

```html
<!-- 実行されない -->
<script type="module" src="/foo.js"></script>
<script type="module">console.log('Hi!')</script>
```

ページの DOM は以下のようになります。`script` 要素が DOM に追加されているのにもかかわらず、スクリプトは実行されていません。不思議ですね。

![](https://storage.googleapis.com/zenn-user-upload/9834b4deeb55-20251105.png =550x)

比較のために、`type="module"` をつけないバージョンでも見てみましょう。

```html
<!-- 実行される -->
<script src="/foo.js"></script>
<script>console.log('Hi!')</script>
```

DOM 上では余分な属性がいろいろついていますが、こちらは普通にスクリプトが実行されます。

![](https://storage.googleapis.com/zenn-user-upload/dab9299af7ee-20251105.png)

状況をまとめると、こうなります。

- `type="module"` の有無にかかわらず、DOM 上には `script` 要素は追加されている
- ただし、`type="module"` なスクリプトは実行されない

うーん……なぜ `type="module"` だと実行されないのか、これだけだとよくわかりませんね 🤔

## GTM の内部処理を追いかけてみる

よくわからないので、もうちょっと GTM の中まで見てみましょう。ちなみに、GTM の「カスタム HTML」は以下のような仕組みになっています。

1. ユーザーが Web ページを開く
2. HTML に貼り付けた [GTM のスニペット](https://support.google.com/tagmanager/answer/14847097?hl=ja&ref_topic=15191151&sjid=14863096091500092122-NC)によって、GTM が初期化される
3. タグのトリガーが発動すると、GTM によって「カスタム HTML」の内容が HTML に追加される
4. （それによって、カスタム HTML に含まれている `script` 要素の内容が実行される）

デバッガで動きを見てみたところ、`script` 要素を DOM に追加する処理を見つけました。

![](https://storage.googleapis.com/zenn-user-upload/b1b3faad9cf3-20251107.png)

ざっくり説明すると、「カスタム HTML」の内容は配列 `b` に入っていて、これを順次 `e`（= `g`）に取り出して処理していきます。`g` が `script` 要素だった場合、赤く囲った `a.insertBefore(g, null)` によって `a`（`body` 要素）に追加されます[^1]。

[^1]: 簡単のためにはしょっていますが、厳密には、この `a.insertBefore(g, null)` で追加が行われるのは **`src` 属性がない場合**だけです。`src` 属性がある場合、`Mc` という関数の中で似たような処理が行われ、`appendChild()` で `body` 要素に追加されます。

ただよく見ると、「`g` が `script` 要素だった場合」の if 文に、`g.type === "text/gtmscript"` という条件もついています。これについてはちょっと説明が必要ですね。GTM の「カスタム HTML」では、`type` 属性のない `script` 要素は、GTM 側で `type="text/gtmscript"` という属性がつけられた上で、上記引用部分の処理に流れてきます。そのため、`g.type === "text/gtmscript"` も `true` になります。

一方、`type="module"` である `script` 要素の場合は、`type="text/gtmscript"` という属性はつきません。そのため、この条件 `g.type === "text/gtmscript"` は `false` となり、別の枝の方にある `a.insertBefore(g, null)` によって `a`（`body` 要素）に追加されます。

![](https://storage.googleapis.com/zenn-user-upload/43bfa4687188-20251106.png)

まとめるとこうですが、まだ違いはよく分かりませんね。

- A. `<script src="/foo.js">` の場合
  1. `type` 属性がつけられて `<script type="text/gtmscript" src="/foo.js">` になる
  2. 上の方の `a.insertBefore(g, null)` で DOM に追加
  3. （スクリプトが実行される）
- B. `<script type="module" src="/foo.js">` の場合
  1. `type` 属性は変わらずに `<script type="module" src="/foo.js">` のまま
  2. 下の方の `a.insertBefore(g, null)` で DOM に追加
  3. （スクリプトは実行されない）

A.2. の「上の方の `a.insertBefore(g, null)`」の直前にも処理があるので、もうちょっと見てみます。どうやら、`script` 要素をわざわざ作り直してから、`body` に追加しているようです。

![](https://storage.googleapis.com/zenn-user-upload/54be035f6425-20251107.png)

ということで、この作り直しの有無によって、スクリプトが実行されるか否かの違いが出ていそうです。

- A. `<script src="/foo.js">` の場合
  1. `type` 属性がつけられて `<script type="text/gtmscript" src="/foo.js">` になる
  2. **`script` 要素を作り直す**
  3. 上の方の `a.insertBefore(g, null)` で DOM に追加
  4. （スクリプトが実行される）
- B. `<script type="module" src="/foo.js">` の場合
  1. `type` 属性は変わらずに `<script type="module" src="/foo.js">` のまま
  2. 下の方の `a.insertBefore(g, null)` で DOM に追加
  3. （スクリプトは実行されない）

今度は逆に、もう少し上流を追いかけてみます。先ほど

> ざっくり説明すると、「カスタム HTML」の内容は配列 `b` に入っていて、これを順次 `e`（= `g`）に取り出して処理していきます。`g` が `script` 要素だった場合、赤く囲った `a.insertBefore(g, null)` によって `a`（`body` 要素）に追加されます。

と書きましたが、この配列 `b` の出どころを探ります。すると、以下のような流れで配列 `b` が作られていることが分かりました。

1. `document.createElement('div')` で、`div` 要素を作成しておく
1. ``div.innerHTML = `A<div>${カスタム HTML の内容}</div>`;`` で内容をセット
1. その `div` の子孫から、あらためてカスタム HTML の内容（各要素）を取り出す

回りくどいですね…。これにどんな意図があるのかよく分かりませんが。ただキーポイントとしては、**ここでは `script` 要素を直接作成したわけではない**という点です。

```ts
const div = document.createElement('div');
div.innerHTML = `A<div><script type="module" src="/foo.js"></div>`;
// 以下略
```

のようにして、結果的に `script` 要素ができてはいます。しかし、このように `div.innerHTML` で作成された `script` 要素をそのまま DOM に追加しても、**そのスクリプトの内容は実行されません**。HTML Living Standard でいうと、おそらく[このあたりの話](https://html.spec.whatwg.org/dev/scripting.html#:~:text=or%20HTML%20parsing\).-,When%20inserted%20using%20the,attributes%2C%20they%20do%20not%20execute%20at%20all.,-The)でしょう。

これについては開発者ツールのコンソールで、以下の簡略化したコードを実行してみれば分かります。

```js:div.innerHTML の結果作成された script 要素では、スクリプトは実行されない
const div = document.createElement('div');
div.innerHTML = '<script>console.log("Hi!")</script>';
const script = div.firstChild;
document.body.insertBefore(script, null); // => スクリプトは実行されない
```

```js:script 要素を直接作ったのであれば、スクリプトは実行される
const script = document.createElement('script');
script.innerHTML = 'console.log("Hi!")';
document.body.insertBefore(script, null); // => スクリプトは実行される
```

どちらも DOM 上には `script` 要素が追加されますが、それが実行されるか否かは異なります。

またもちろん、`document.createElement('script')` で `script` 要素を作り直せば、スクリプトは実行されます。

```js
const div = document.createElement('div');
div.innerHTML = '<script>console.log("Hi!")</script>';
const script = div.firstChild;

const script2 = document.createElement('script');
script2.text = script.text
document.body.insertBefore(script2, null); // => スクリプトは実行される
```

ということで、長々と見てきて結局のところそこまで大した話ではないですが、`type="module"` だと GTM ではうまく動かない理由が分かりました。

1. `div.innerHTML` への代入の結果、`script` 要素が作成される
2. `type="module"` をつけていなければ、`script` 要素があらためて作り直される
3. `script` 要素が DOM に追加される
4. （2. で作り直されていれば、スクリプトが実行される）

余談ですが、GTM のこのあたりの処理では [Trusted Types API](https://developer.mozilla.org/en-US/docs/Web/API/Trusted_Types_API) も使われています。ｵｯ! と思ってコードを追いかけてみましたが、今回の問題の本質とは別に関係はなかったようです。

## おわりに

この記事では、`type="module"` な `script` 要素が GTM で実行されない件について見てみました。GTM を使う側も、スクリプトを提供する側も、注意する必要がありそうですね。

ちなみに、GTM とスクリプトの関係を調べている途中で、以下のブログ記事を見つけました。こちらもおもしろいのでおすすめです。

https://techblog.raccoon.ne.jp/archives/1660010900.html
