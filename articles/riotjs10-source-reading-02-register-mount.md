---
title: 'Riot.js 10 のソースコードを読む (2) register と mount を繋いでいるもの'
emoji: '🔗'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['riotjs', 'riot', 'javascript', 'ソースコードリーディング']
published: false
---

こんにちは．[Keeth](https://x.com/kuwahara_jsri) こと桑原です．

[Riot.js（以下，riot）](https://riot.js.org/) のソースコードを読むシリーズの 2 回目です．前回は「どこに何があるか」の地図を描きました．

https://zenn.dev/keeth/articles/riotjs10-source-reading-01-overview

前回書いたとおり，`riot/riot` の `src/` は 1081 行しかありません．今回はそのうち 40 行くらいを読みます．`riot.register()` と `riot.mount()` の話です．

:::message
本記事の数字やコードは，`riot/riot` の **v10.1.6**（本記事執筆時点の最新）を対象にしています．
:::

## TL;DR

- `register()` がやっているのは，グローバルな `Map` への `set` だけ．コンポーネントの実体ではなくファクトリ関数を入れている
- `mount()` はその `Map` から `get` するだけで，`register.js` を import していない．import を辿る読み方はここで一度切れる
- プラグインも同じで，`install()` が `Set` に足して，コンポーネント生成時に `reduce` で通す
- `is` 属性とタグ名を同じように扱っているのは `src/utils/dom.js` の 3 行
- `riot.min.js` では `panic()` の呼び出しが terser に消されていて，エラーメッセージが 1 つも残っていない

## セットで使うのに，繋がっていない

`register()` と `mount()` はセットで使います．

```js
import { register, mount } from 'riot'
import MyTag from './my-tag.riot'

register('my-tag', MyTag)
mount('my-tag')
```

なのでコードの上でも繋がっているだろうと思って読み始めたら，繋がっていませんでした．そこで手が止まったので，そこから書きます．

## register の中身

`src/api/register.js` は，JSDoc のコメントを除くと 11 行です．

```js
export function register(name, { css, template, exports }) {
  if (COMPONENTS_IMPLEMENTATION_MAP.has(name))
    panic(`The component "${name}" was already registered`)

  COMPONENTS_IMPLEMENTATION_MAP.set(
    name,
    createComponentFromWrapper({ name, css, template, exports }),
  )

  return COMPONENTS_IMPLEMENTATION_MAP
}
```

やっていることは `Map` への `set` だけです．登録した時点ではコンポーネントのインスタンスは作られません．作られるのはファクトリ関数で，中身は `src/core/create-component-from-wrapper.js` がこう返しています．

```js
return ({ slots, attributes, props }) => {
  // ...
}
```

つまり `register()` が `Map` に入れているのは「props と slots を渡すとコンポーネントになる関数」であって，コンポーネントそのものではありません．だから 1 回の `register` で，そのタグを何百個マウントしても問題がないわけです．

`COMPONENTS_IMPLEMENTATION_MAP` は `@riotjs/util` から import しています．ここが引っかかりどころで，ユーティリティのパッケージにアプリケーションの状態が入っています．実体は `constants.js` にある，ただの `new Map()` です．

```js
export const COMPONENTS_IMPLEMENTATION_MAP = new Map(),
  DOM_COMPONENT_INSTANCE_PROPERTY = Symbol('riot-component'),
  PLUGINS_SET = new Set(),
  IS_DIRECTIVE = 'is',
  // ...
```

## mount の中身

`src/api/mount.js` も，コメントを除けば 5 行です．

```js
export function mount(selector, initialProps, name) {
  return $(selector).map((element) =>
    mountComponent(element, initialProps, name),
  )
}
```

`$` は `bianco.query` で，`querySelectorAll` の結果を配列にするだけのものです．`mount()` が配列を返すのはこの `map` のせいで，セレクタが 1 個しかマッチしなくても配列で返ってきます．

実際の仕事は `src/core/mount-component.js` にあります．

```js
export function mountComponent(element, initialProps, componentName, slots) {
  const name = componentName || getName(element)
  if (!COMPONENTS_IMPLEMENTATION_MAP.has(name))
    panic(`The component named "${name}" was never registered`)

  const component = COMPONENTS_IMPLEMENTATION_MAP.get(name)({
    props: initialProps,
    slots,
  })

  return component.mount(element)
}
```

ここで手が止まりました．このファイルの import 文は 2 行だけです．

```js
import { COMPONENTS_IMPLEMENTATION_MAP, panic } from '@riotjs/util'
import { getName } from '../utils/dom.js'
```

`register.js` を参照していませんし，`create-component-from-wrapper.js` も参照していません．コンポーネントの実装がどこから来るのかというと，`Map` から `get` しているだけです．

## 依存グラフに現れない辺

`src/` の import を全部拾って図を描くと，`register.js` と `mount-component.js` のあいだには線が 1 本も引けません．でも実行時には，前者が書いたものを後者が読んでいます．

```
api/register.js  --set-->  COMPONENTS_IMPLEMENTATION_MAP  --get-->  core/mount-component.js
                                 (@riotjs/util)
```

読み手として言うと，これはやりづらいです．import を辿って理解する読み方が，ここで一度切れます．自分は `grep -rn COMPONENTS_IMPLEMENTATION_MAP src` を打つまで繋がりが見えませんでした．

書き手の側から見ると，この形にする理由は分かります．登録とマウントを完全に別のフェーズにできるからです．`register()` は DOM が存在しなくても呼べますし，`mount()` はコンポーネントの実装を知らなくても呼べます．`.riot` ファイルを個別に import して個別に register していく使い方が成立するのは，この `Map` があるおかげです．

同じパターンがもう 1 箇所あります．プラグインです．

```js
// src/api/install.js
PLUGINS_SET.add(plugin)

// src/core/run-plugins.js
return [...PLUGINS_SET].reduce((c, fn) => fn(c) || c, component)
```

`install()` が `Set` に足して，全コンポーネントの生成時に `reduce` で通しています．これも import では繋がっていません．

ここで少し面白いのは，riot が lint で関数型のルールを強制していることです．`eslint.config.js` の `rules` はこうなっています．

```js
rules: {
  'fp/no-rest-parameters': 0,
  'fp/no-mutating-methods': 0,
  'jsdoc/no-undefined-types': 0,
},
```

`fp/` から始まるルールは [eslint-plugin-fp](https://github.com/jfmengels/eslint-plugin-fp) のもので，`eslint-config-riot` の peerDependencies に入っています．ここに書いてあるのは「無効にしたルール」のリストなので，残りの no-mutation 系のルールは生きているということです．実際 `src/` の 45 ファイルに `class` は 1 つもなくて，`compose`（`cumpa`）と `curry`（`curri`）と `reduce` で組み立てられています．

そうやって純粋関数を並べる一方で，コンポーネントの所在は書き換え可能なグローバル `Map` で解決しています．矛盾というほどではないですが，ちぐはぐではあります．自分はどちらかというと，`Map` のほうが実用的な判断だと思っています．これを関数型でやろうとすると，登録済みコンポーネントの集合をアプリケーション全体で持ち回ることになって，`riot.mount('my-tag')` という API が書けなくなります．

## 名前の解決

`mountComponent` の 1 行目に戻ります．

```js
const name = componentName || getName(element)
```

`getName` は `src/utils/dom.js` にあって，実体は 3 行です．

```js
export function getName(element) {
  return getAttr(element, IS_DIRECTIVE) || element.tagName.toLowerCase()
}
```

`IS_DIRECTIVE` は `@riotjs/util` にある文字列の `'is'` です．つまり `is` 属性があればそれを，なければタグ名を小文字にしたものをコンポーネント名として使います．

```html
<my-tag></my-tag>
<div is="my-tag"></div>
```

この 2 つが同じように動くのは，この 3 行のおかげです．そして riot が内部管理のために DOM へ書き込む属性も，この `is` だけです（`src/core/add-css-hook.js` がやっています）．`data-` 属性も管理用のクラスも足しません．親子関係やテンプレートの参照は Symbol でインスタンス側に隠してあります．

## minify すると消えるもの

ここまでに `panic()` が 2 回出てきました．未登録のコンポーネントをマウントしたときと，同じ名前で二重に register したときのエラーです．

この 2 つのメッセージを `riot.js` と `riot.min.js` で検索すると，結果が違います．

```bash
$ grep -o "was already registered\|never registered" riot.js
was already registered
never registered
never registered

$ grep -o "was already registered\|never registered" riot.min.js
# 1 行も出てこない
```

`riot.js` の側で `never registered` が 2 回出るのは，`unregister()` が同じ文言を使っているからです．そして minify 版では 3 つとも消えています．

理由は Makefile の minify オプションに書いてあります．

```
MINIFY_OPTIONS = --comments false \
				 --toplevel \
				 --compress pure_funcs=['panic'],unsafe=true,unsafe_symbols=true,passes=5 \
				 --mangle \
```

`pure_funcs=['panic']` は terser に「この関数は副作用がないので，戻り値を使っていない呼び出しは消していい」と伝えるものです．`panic` の実体は `@riotjs/util` の `misc.js` にあって，実際には throw します．

```js
export function panic(message, cause) {
  throw new Error(message, { cause })
}
```

消すと挙動が変わるわけですが，実際に消えています．`riot.min.js` の `mount` は次のようになっていました（1 行なので読みやすさのために整形しています）．

```js
t.mount = function (t, e, n) {
  return $t(t).map((t) =>
    (function (t, e, n) {
      const r = n || Xt(t)
      return u.has(r), u.get(r)({ props: e, slots: void 0 }).mount(t)
    })(t, e, n),
  )
}
```

`if (!u.has(r)) panic(...)` が `u.has(r),` という何もしない式になっています．`panic` の呼び出しだけを terser が落として，条件式が残骸として残った形です．

結果として，minify 版で未登録のコンポーネントをマウントすると，`The component named "my-tag" was never registered` ではなく，`u.get(r)` が返した `undefined` を呼び出したときの `TypeError` が出ます．開発者向けのメッセージを丸ごと捨てて 15,856 バイトを守っているわけです．

React のように dev ビルドと prod ビルドを分ける手もあったはずで，そうしなかったのは判断だと思います．ただ，デバッグする側としては正直きついです．`riot.min.js` を本番に置いていて原因不明の `TypeError` が出たら，まず `riot.js` に差し替えて再現させたほうが早いでしょう．

ついでに言うと，`panic` は 1 行の `throw new Error` です．Go の `panic` みたいな名前をしていますが，recover するような仕組みはありません．なぜ 1 行の関数を別パッケージに置いているのかというと，`@riotjs/util` が 4 リポジトリの共有置き場だからという説明はつきます．ただ，`throw` を 1 箇所に集める実利はあまり感じないので，terser に名前で指定するために関数の形にしておきたかった，という理由のほうが納得できます．

## 次回

`COMPONENTS_IMPLEMENTATION_MAP.get(name)({ props, slots })` が返すオブジェクトの中身を見ます．`src/core/manage-component-lifecycle.js` の 174 行で，前回「ここだけ読めば 9 割分かる」と書いたファイルです．

https://zenn.dev/keeth/articles/riotjs10-source-reading-03-lifecycle
