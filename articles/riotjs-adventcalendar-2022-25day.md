---
title: "2022 年の JS 界隈を振り返りつつ Riot.js の今後についてを軽く妄想する"
emoji: "👏"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["riotjs", "ポエム", "アドベントカレンダー", "adventcalendar", "JavaScript"]
published: false
---

新年明けましておめでとうございます←

本記事は [Riot.js Advent Calendar 2022](https://qiita.com/advent-calendar/2022/riotjs) の __第25日（トリ）に投稿しようとした__ 記事となります．「もう新年明けてもうてるやないかい！」というツッコミはありがたく頂戴致します🙇

また本記事はポエム調ですので悪しからず．

# 2022 年に流行した JS ライブラリ

2022 年の終わりに 2021 年の [State of JS 2021](https://2021.stateofjs.com/en-US/libraries/front-end-frameworks/#front_end_frameworks_experience_ranking) アンケート結果を見るのもどうかと思いますが，個人的にはインパクトがあったので引用させていただきます．

昨年は以下の４つの観点

> Technologies with less than 10% awareness not included. Each ratio is defined as follows:
>
>
> ・Retention: would use again / (would use again + would not use again)
>
> ・Interest: want to learn / (want to learn + not interested)
>
> ・Usage: (would use again + would not use again) / total
>
> ・Awareness: (total - never heard) / total


で集計されていました．記載の通り，認知度が10％未満の技術は載っていないそうですね．


![Retention](https://storage.googleapis.com/zenn-user-upload/bd41121e79a1-20230111.png)

今まで聞いたことがなかった [Solid.js](https://www.solidjs.com/) というライブラリが１位に躍り出ていたので，新年早々驚きました．公式ドキュメントを読んで見ると

* `React` ライクなシンタックス（`React Hooks` のような機能）
* コストのかかる仮想DOMを用いず，ハイパフォーマンス
* チュートリアルやドキュメントが充実

パフォーマンスに関する根本思想については [こちらの記事](https://ryansolid.medium.com/solidjs-the-tesla-of-javascript-ui-frameworks-6a1d379bc05e) を見てみてください．

:::details 触ってみた感想

軽く触ってみたところ，確かに React ライクだなと強く感じました．それでいて開発・実行パフォーマンスもよく，今のところ体験はとても良いです．React Hooks，例えば `useState()` に当たるものが Solid.js では `createSignal` という名前で存在します．

参考程度に [公式チュートリアル](https://www.solidjs.com/tutorial/introduction_signals?solved) のコードを転載します．

```jsx:公式のサンプルコード
import { render } from "solid-js/web";
import { createSignal } from "solid-js";

function Counter() {
  const [count, setCount] = createSignal(0);

  setInterval(() => setCount(count() + 1), 1000);

  return <div>Count: {count()}</div>;
}

render(() => <Counter />, document.getElementById('app'));
```

公式サイトでも
`Vite` を用いた [様々なテンプレート](https://github.com/solidjs/templates) が用意されていたり，`Next.js` のようなラッパー [degit](https://github.com/Rich-Harris/degit) も用意されているなど，入門もしやすいですね．
:::

![Interest](https://storage.googleapis.com/zenn-user-upload/5589c2ca40a8-20230111.png)
![Usage](https://storage.googleapis.com/zenn-user-upload/91cd23be38cf-20230111.png)
![Awareness](https://storage.googleapis.com/zenn-user-upload/bb47d3ec157e-20230111.png)

（以上４画像はこちらから引用）

https://2021.stateofjs.com/en-US/libraries/front-end-frameworks/#front_end_frameworks_experience_ranking


# 実際の DL 数比較

実際の DL 数を npm の API をコールして見ました．

```sh
$ csm -t react vue @angular/core angular svelte solid-js remix astro @builder.io/qwik

╔═══════════╤════════════╤════════════╤══════════════════╗
║ downloads ║ start      ║ end        ║ package          ║
╟───────────┼────────────┼────────────┼──────────────────╢
║ 742231548 ║ 2022-01-01 ║ 2022-11-29 ║ react            ║
╟───────────┼────────────┼────────────┼──────────────────╢
║ 154835300 ║ 2022-01-01 ║ 2022-11-29 ║ vue              ║
╟───────────┼────────────┼────────────┼──────────────────╢
║ 140478869 ║ 2022-01-01 ║ 2022-11-29 ║ @angular/core    ║
╟───────────┼────────────┼────────────┼──────────────────╢
║ 60448440  ║ 2022-01-01 ║ 2022-11-29 ║ svelte           ║
╟───────────┼────────────┼────────────┼──────────────────╢
║ 24637946  ║ 2022-01-01 ║ 2022-11-29 ║ angular          ║
╟───────────┼────────────┼────────────┼──────────────────╢
║ 1390052   ║ 2022-01-01 ║ 2022-11-29 ║ solid-js         ║
╟───────────┼────────────┼────────────┼──────────────────╢
║ 874394    ║ 2022-01-01 ║ 2022-11-29 ║ remix            ║
╟───────────┼────────────┼────────────┼──────────────────╢
║ 834456    ║ 2022-01-01 ║ 2022-11-29 ║ astro            ║
╟───────────┼────────────┼────────────┼──────────────────╢
║ 87878     ║ 2022-01-01 ║ 2022-11-29 ║ @builder.io/qwik ║
╚═══════════╧════════════╧════════════╧══════════════════╝
```
