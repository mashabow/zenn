---
title: "乾太くんで家庭内感染が防げるか ChatGPT にシミュレーションしてもらう"
emoji: "👕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["chatgpt", "シミュレーション"]
published: true
publication_name: "socialplus"
---

3 歳になった子どもが体調不良で嘔吐しました。パジャマも布団カバーもべちゃべちゃです。おそらく風邪だとは思いますが、ノロウイルスなんかに家庭内感染するのも嫌なので、念のためハイターでつけ置き洗いをします。子どもの体をシャワーで洗って、新しいパジャマを着せて、寝かしつけをしている間にふと思いました。

つけ置き洗いが不十分だったとしても、乾太くんで乾かせばある程度除菌できたりするんだろうか…？ 🤔

子どもが腕に乗っていて動けないので、スマホ片手に ChatGPT に聞いてみます。最近また賢くなった気がしますが、どこまで答えてくれるんでしょうか。

:::message alert
この記事のシミュレーション結果を信用しないでください。これが果たして妥当なのか、今回は何も検証していません。また、筆者は専門家でも何でもない素人です。
:::

:::message
つけ置き洗いをしないと洗濯機から感染拡大するリスクがあります。
:::

## シミュレーションを出すまでのやりとり

GhatGPT 4o とのやりとりを抜粋して紹介していきます。ちなみに、別に最初から「シミュレーションしてもらおう」と考えていたわけではなくて、話をしていくうちに「シミュレーションしないと Yes か No かよくわからないから、お願いするか」となったのが実際のところです。

なお、やりとりの全文ついては以下のリンク先をご覧ください。

https://chatgpt.com/share/6804416d-3c8c-8011-8a62-b5b9fe2f9419

まずさらっと聞いてみます。

![](https://storage.googleapis.com/zenn-user-upload/2b07d1b826b8-20250423.png)

基本は押さえているようです。ノロウイルスの場合は、酸素系漂白剤ではなく塩素系漂白剤を使うべきですが。

![](https://storage.googleapis.com/zenn-user-upload/245541884406-20250423.png)

乾太くんという固有名詞も出てきました。「85℃ 以上で 1 分加熱すれば不活化」とありますが、乾太くんの温度や時間で十分なのか気になります。

何往復かやりとりをしたあとに「衣類の表面温度は 80℃ 近くになる」と言ってきましたが、どこまで本当かわからないので、**ソースを出せ**と問い詰めます。

![](https://storage.googleapis.com/zenn-user-upload/97e8be09bca1-20250423.png)

[東京ガスのページ](https://home.tokyo-gas.co.jp/housing/other/kanta/index.html)を出してきました。見てみると、ちゃんと乾太くんの実際の温度のグラフがあります 👍️

> ![](https://storage.googleapis.com/zenn-user-upload/298fa6de5939-20250423.png)

ただ、このデータだと 72℃ あたりがピークでなので、「80℃ 近くになる」はちょっと言いすぎですね。あとは、「温度と時間によってノロウイルスがどれだけ不活化するか」のデータがあれば、本当に感染対策になるのかざっくり計算できそうです。

![](https://storage.googleapis.com/zenn-user-upload/592549378607-20250423.png)

検索してくれた画像は的外れなものが多く、精度はいまいちでしたが、質問自体は理解できているようです。参考になりそうな論文を引っ張ってきてくれました。

自分で論文を探そうとしても、門外漢では当たりすらつけられないので、こういう面はとても助かりますね。解説もしてくれますし、わからない部分を質問すればどんどん答えてもらえます。

![](https://storage.googleapis.com/zenn-user-upload/dfef826fcb4b-20250424.png)

Weibull モデル（初耳）についても、完全に理解した気になれました。この Weibull モデルを使えば、「ある温度である時間放置したとき、どれぐらいウイルスが生き残っているか？」が分かるそうです。

![](https://storage.googleapis.com/zenn-user-upload/edd6b9947a2b-20250424.png)

ちょっと歯ごたえのある説明ですが。最近、ChatGPT の**カスタム指示**に以下のように書いてみたので、このような答えを返してくれるのかもしれません。

> ユーザーは工学部出身で、文系理系を問わず知的好奇心をもっています。専門のコンピュータサイエンス以外は忘れてしまった知識も多いですが、背景にある基礎的な概念を理解したいと考えています。
>
> 2022 年生まれの息子がおり、育児上の相談をするかもしれません。この場合、学術的なエビデンスや公的機関の見解をある程度重視してください。

さて、「生存率をグラフにして可視化できますよ」と向こうから言ってくれたので、お願いしてみます。気が利きますね。

![](https://storage.googleapis.com/zenn-user-upload/51343b567742-20250424.png)

残念ながら、日本語の文字が豆腐になっています。「日本語フォントがインストールされていないんじゃ？」と教えてあげたのですが、解消できなかったようなのでとりあえず英語で進めました。

ただし、こういったモデルのパラメータは、かなり適当に決めて来がちな印象があります。**このパラメータを選んだ根拠は？** と問いただしてやると、考え直して修正してくれたりします。

![](https://storage.googleapis.com/zenn-user-upload/c60551463a25-20250424.png)

1 つ前のグラフとは全然違う結果になりました。

ここまで来たら、先ほどの乾太くんの温度データと、この Weibull モデルを突き合わせれば、乾太くんをかけた後のノロウイルスの生存率が計算できるはずです。

![](https://storage.googleapis.com/zenn-user-upload/c5a4f65321e2-20250424.png)

[先ほどの東京ガスのページ](https://home.tokyo-gas.co.jp/housing/other/kanta/index.html)の URL を渡して、グラフから温度データを読み取ってほしかったのですが、うまく読み取ってくれませんでした。グラフ画像の URL を渡しても同様です。結局、**グラフの画像ファイルを添付**したら読み取りに成功しました。温度のデータが抽出できています。

![](https://storage.googleapis.com/zenn-user-upload/acd92033638e-20250424.png)

これでデータが揃ったので、あとはシミュレーションする（してもらう）だけです。

![](https://storage.googleapis.com/zenn-user-upload/e279410e30ac-20250424.png)

それっぽいグラフが出てきました。シミュレーションに使ったコードを確認できるのもおもしろいですね。

```py
# 温度推移（読み取ったおおよその温度データ）[℃]
minutes = np.array([0, 10, 20, 30, 35, 40, 45, 48, 50, 55])
temps = np.array([30, 38, 39, 40, 45, 55, 63, 70, 65, 60])

# 時間軸（1分刻み）
t_minutes = np.arange(0, 56, 1)
t_seconds = t_minutes * 60  # 秒に変換

# 線形補間で温度データを毎分生成
interpolated_temps = np.interp(t_minutes, minutes, temps)

# Weibull不活化モデル：log10(N/N0) = - (t/δ(T))^p
# δ(T) = δ60 * 10^((60 - T)/z)
z = 11
delta60 = 316  # Bozkurtらの報告値（60℃でのδ）
p = 1.2

# δを各時点の温度から算出
log_delta_t = np.log10(delta60) + (60 - interpolated_temps) / z
delta_t = 10 ** log_delta_t

# 不活化率（毎分積算）: log10(N(t)/N0) = -∑[(Δt/δ)^p]
dt = 60  # 秒（1分刻み）
log_reduction = np.zeros_like(t_minutes, dtype=float)

for i in range(1, len(t_minutes)):
    d = delta_t[i]
    log_reduction[i] = log_reduction[i-1] - (dt / d) ** p

survival_ratio = 10 ** log_reduction
```

こちらから聞けば、シミュレーション手法の解説もしてくれます。

![](https://storage.googleapis.com/zenn-user-upload/8159f2859b67-20250424.png)

最後に、グラフの体裁をちょっと整えたり、似たようなリスクのある他のウイルス（これも ChatGPT に挙げてもらいました）でもシミュレーションしてもらったりして、完成です 👏

![](https://storage.googleapis.com/zenn-user-upload/bc4b173a0e6a-20250424.png)

これによると、ノロウイルスやロタウイルスは乾太くんでほぼゼロになりますが、アデノウイルスは数%残ってしまうようです。

:::message alert
このシミュレーション結果を信用しないでください。これが果たして妥当なのか、今回は何も検証していません。また、筆者は専門家でも何でもない素人です。
:::

## ChatGPT の他のモデルでは？

ついでに、他のモデルだとどんな感じなのか、o3, o4-mini, o4-mini-high でざっと試してみました。今回のテーマだと、そこまで能力の差は実感できなかったというのが正直なところです。それよりも、**レスポンスの速い 4o の方が、こういった探索的なやりとりには向いている**印象です。

どのモデルも、温度の推移のグラフの読み取りに苦戦していました。ページの URL を渡せば縦軸・横軸は理解（推測？）してくれますが、グラフから読み取った数値は大きな誤差があり、使い物になりませんでした。グラフの画像ファイルを添付すれば読み取れましたが、o4-mini だけはどう助け舟を出しても読み取ることができませんでした。ちなみに、o3 は画像解析の過程もつぶやいてくれるので、見ていてなかなかおもしろいです。

:::details o3 の画像解析のステップ（一部）

画像上の座標と軸の対応関係の推定：

![](https://storage.googleapis.com/zenn-user-upload/5555486380eb-20250424.png)

スムージング：

![](https://storage.googleapis.com/zenn-user-upload/7e3bca5d89f8-20250424.png)

:::

また、乾太くんの「80 ℃ 以上の高温乾燥」の謳い文句に引きずられて、「衣類の温度もスタート直後から 80 ℃ 以上になる」という勘違いも共通していました。o4-mini-high は、水分の蒸発を考慮して衣類温度の推移を計算しようとしましたが、洗濯機による脱水後の水分量ではなく、乾いた服の水分量を使っていました。おしい。

## おわりに

未知の分野にもかかわらず、1 時間足らずでそれっぽいシミュレーションができて、なかなかおもしろい体験でした。4o はレスポンスも速く、また「こうしましょうか？」「これもできますよ？」と次の手を聞いてきてくれて、さくさくテンポ良く話が膨らみます。

こういったシミュレーションでは、いろいろな仮定をおいてパラメータを設定する必要がありますが、全般的に、結構適当な値を使ってくる印象です。こちらから与える情報が少ないので、仕方ないといえば仕方ないんですけども。ちゃんと使おうと思ったら、パラメータが妥当なのかチェックしてやる必要がありますね。また、シミュレーションの手法も果たしてこれで良いのか、検証が必要です。

## おまけ：一般人向けのわかりやすい画像

「シミュレーション結果を元に、一般人向けのわかりやすい画像を作って」と頼んだら、データを捏造してきました 😡

![](https://storage.googleapis.com/zenn-user-upload/592ebb710370-20250424.png)

再提出された画像もめちゃくちゃです。まぁ、こういう画像生成は単純に向いていないんでしょうね。

![](https://storage.googleapis.com/zenn-user-upload/cc4c0334d41a-20250424.png)

最後の 1 文の気遣いがなんとも…。いや、ありがたいはありがたいんですが。
