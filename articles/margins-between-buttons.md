---
title: '複数の button タグの間に謎の余白が作られる'
emoji: '💠'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['css', 'tips', 'スタイリング']
published: true
---

こんにちは．[株式会社ゆめみ](https://www.yumemi.co.jp/) の Keeth こと桑原です．

<s>今回の内容はフロントエンドエンジニアであればご存じの方も多いかもしれませんが，お恥ずかしいことに私は知らなかったので備忘録として書き残したいと思います．</s>

<b>2024/11/06 追記</b>

これは単に `<button>` タグがインライン‐ブロック要素であり，以下の Codepen のコードのように，改行や空白など HTML タグ間に余白があると画面にもそのように表示されているだけでした．

## 発生した事象

タイトル通りと言えばそうなんですが，百聞は一見にしかずということで，Codepen のデモを貼ります 👇

@[codepen](https://codepen.io/kuwahara_jsri/pen/wvgNwvx)

ご覧いただくと分かるように，`border, padding` を 0 で初期化したとしても，余白が設定されていることが分かるかと思います．この余白を消す方法としては以下の方法が考えられます．

```css
div {
  display: flex;
}

/* または */

div {
  font-size: 0;
}
```

ちなみに， `border, background` の設定を外すと，ブラウザ固有のスタイリング（今回は `border`）が設定されるため，余白が生じますのでご注意ください．

![各ブラウザの挙動2](https://storage.googleapis.com/zenn-user-upload/a59978002f1tzc23mpo84j0ycwb3)

ではでは(=ﾟ ω ﾟ)ﾉ
