---
title: 'Riot.js 5.2.0 がリリース🎉 旧バージョンの文法もサポート'
emoji: '🔥'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['riotjs', 'JavaScript', 'update']
published: true
---

つい先日 [Riot.js(以下、riot)](https://riot.js.org) のバージョン v5.2.0 がリリースされました ❗ このバージョンのリリースは riot 界隈でちょっと話題になっており、今回はこのアップデートについて書き残していきます。

https://twitter.com/riotjs_/status/1355974386721943554?s=20

# リリースノート

公式リポジトリのリリースノートの [v5.2.0](https://github.com/riot/riot/releases/tag/v5.2.0) を見てみると、以下のように書かれていました。

> Some liked more the old RIot.js syntax so you can now write components also as follows

なるほど、思ったより v3 系の昔のシンタックスが人気だったようですねー。v4.0.0 にて破壊的変更が入り、シンタックスもドラスティックに変更が入りましたので、v4 系から riot から離れた方も一定数いると思います。

では次にリリースノートに載せてあるコードを見てみましょう。

```html
<old-syntax>
  <p>{ state.message }</p>
  <button onclick="{onClick}">Click Me</button>

  <script>
    this.onBeforeMount = () => {
      this.state.message = 'Hello'
    }

    this.onClick = () => {
      this.update({
        message: 'Goodbye',
      })
    }
  </script>
</old-syntax>
```

なるほど。`export default {}` を書かなくても良くなったようですね。ちろんですが書いても動作することは確認しました。ちなみに現在の書き方は以下のようになりますので、比較のために書いておきます。

```html
<current-syntax>
  <p>{ state.message }</p>
  <button onclick="{onClick}">Click Me</button>

  <script>
    export default {
      onBeforeMount() {
        this.state.message = 'Hello'
      },
      onClick() {
        this.update({
          message: 'Goodbye',
        })
      },
    }
  </script>
</current-syntax>
```

さすがにこれだけだとわからないので、もう少しどのような書き方が行けるのか見ていきます 👍

## ３つの引数：opts, props, state

まずは３つの引数 `opts, props, state` の確認から。以下のコードにて確認しましたところ、`props, state` は問題なく動作しましたが、`opts` はエラーになったため、やはり使えないようです。

```javascript
console.log(opts) // opts is not defined

this.onBeforeMount = (props, state) => {
  console.log(props) // {title: "Old Syntax Demo"}
  state.message = 'Hello'
}
```

## ライフサイクルメソッドの書き方

次にライフサイクルの書き方です。こちらも現在の書き方は行けましたが、旧バージョンの書き方である `this.on('xxx')` は使えないようです。

```javascript
// OK
this.onMounted = () => {
  this.update({
    message: 'Bye',
  })
}

// NG
// SyntaxError: Unexpected token
this.on('mount', () => {
  console.log(this)
})
```

## script タグの省略

旧バージョンの書き方では、`<script>` タグを書かなくても良かったのですが、こちらは省略不可能でした。

```html
<old-syntax>
  <p>{ state.message }</p>
  <button onclick="{onClick}">Click Me</button>

  <!-- error -->
  this.onBeforeMount = () => { this.state.message = 'Hello' } this.onClick = () => {
  this.update({ message: 'Goodbye', }) }
</old-syntax>
```

## 旧バージョンの機能は使えない

`riot.mount('*')` のように mount メソッドのワイルドカードも、`observable, mixin` などももちろん使用できません。

# 終わりに

正直期待値をちょっと下回った、というのが本音になります 😅 v3 系のシンタックスをサポートするという事前の告知から、色々復刻すると思ってしまいましたがそれは早計でしたね 💦 言ってしまえば、`export default {}` を明示的に書かなくても良くなった、くらいのアップデートでした。それならば、個人的には `export default {}` を明示的に書いたほうが可読性高いので好きですね。

この先のアップデートで、もしかしたら復活する機能もあるかもしれませんので、淡い期待をしておきつつ本記事を締めさせていただきます w （本音を言えば mixin がほしいなぁ）

ではでは(=ﾟ ω ﾟ)ﾉ
