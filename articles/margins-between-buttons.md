---
title: '複数の button タグの間に謎の余白が作られる'
emoji: '💠'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['css', 'tips', 'スタイリング']
published: true
---

こんにちは．[株式会社ゆめみ](https://www.yumemi.co.jp/) の Keeth こと桑原です．

今回の内容はフロントエンドエンジニアであればご存じの方も多いかもしれませんが，お恥ずかしいことに私は知らなかったので備忘録として書き残したいと思います．

## 発生した事象

タイトル通りと言えばそうなんですが，百聞は一見にしかずということで，Codepen のデモを貼ります 👇※見やすさのため多少色を変更しています．

@[codepen](https://codepen.io/kuwahara_jsri/pen/wvgNwvx)

ご覧いただくと分かるように，`border, padding, border` をすべて 0 で初期化したとしても，余白が設定されていることが分かるかと思います．この余白を消す方法の一つとして，今回は `<button>` タグを `<div>` タグでラップし `display: flex` を付けて対処しました．

## 各ブラウザ毎の挙動

当方のメインブラウザは Google Chrome となっておりますので，比較のため別ブラウザも見てみたいと思います．こちらは画面のキャプチャとなりますので，ご了承ください．

![各ブラウザの挙動1](https://storage.googleapis.com/zenn-user-upload/onleo3kytys0ji8bg44vub70rt84)

概ね全ブラウザ問題なさそうですね．なお，Edge については Chronium ベースとなったため，Chrome で代用とします．

ちなみに，`border, background` の設定を外すと，Firefox, Safari ともにブラウザ固有のスタイリング（今回は `border`）が設定されるため，余白が生じますのでご注意ください．

![各ブラウザの挙動2](https://storage.googleapis.com/zenn-user-upload/a59978002f1tzc23mpo84j0ycwb3)

ではでは(=ﾟ ω ﾟ)ﾉ
