---
title: 'Riot.js 10 のソースコードを読む (3) 174 行の心臓部'
emoji: '⚙️'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['riotjs', 'riot', 'javascript', 'ソースコードリーディング']
published: false
---

こんにちは．[Keeth](https://x.com/kuwahara_jsri) こと桑原です．

[Riot.js（以下，riot）](https://riot.js.org/) のソースコードを読むシリーズの 3 回目です．前回は `register()` と `mount()` が `Map` 経由で繋がっていることを見ました．

https://zenn.dev/keeth/articles/riotjs10-source-reading-02-register-mount

前回の最後で，`COMPONENTS_IMPLEMENTATION_MAP.get(name)({ props, slots })` が返すオブジェクトを見る，と書きました．そこに至るまでに関数が 3 段重なっているので，まずそこをほどきます．

:::message
本記事の数字やコードは，`riot/riot` の **v10.1.6**（本記事執筆時点の最新）を対象にしています．
:::

## TL;DR

- `src/core/manage-component-lifecycle.js` の 174 行に `mount` / `update` / `unmount` の実体が全部入っている
- ライフサイクルフックのデフォルトは全部 `noop` で埋めてあるので，本体側に存在チェックが 1 つもない
- インスタンスは `Object.create(component)` で作られる．ユーザーが書いたメソッドはプロトタイプ側で共有される
- props は `Object.freeze` される．state は浅くマージするだけで凍結も監視もされない
- `unmount(preserveRoot)` は boolean のはずが，`null` に「HTML に触らない」という第 3 の意味を持たせている

## ファクトリが 3 段ある

`src/core/create-component-from-wrapper.js` の中で，実際にコンポーネントが作られるのはこの 1 行です．

```js
const component = instantiateComponent({
  css,
  template: templateFn,
  componentAPI,
  name,
})({ slots, attributes, props })
```

`instantiateComponent(...)` を呼んで，その戻り値をもう 1 回呼んでいます．戻り値が関数になっているのは，`src/core/instantiate-component.js` の最後がこうなっているからです．

```js
export function instantiateComponent({ css, template, componentAPI, name }) {
  // add the component css into the DOM
  if (css && name) cssManager.add(name, css)

  return curry(manageComponentLifecycle)(
    defineProperties(
      // set the component defaults without overriding the original component API
      defineDefaults(componentAPI, {
        ...COMPONENT_LIFECYCLE_METHODS,
        [PROPS_KEY]: {},
        [STATE_KEY]: {},
      }),
      {
        // defined during the component creation
        [SLOTS_KEY]: null,
        [ROOT_KEY]: null,
        // these properties should not be overriden
        ...COMPONENT_DOM_SELECTORS,
        name,
        css,
        template,
      },
    ),
  )
}
```

`defineDefaults` が「ユーザーが書いていなければ足す」もので，`defineProperties` が「上書きさせない」ものです．実装は `@riotjs/util` の `objects.js` にあって，それぞれこうなっています．

```js
export function defineProperty(source, key, value, options = {}) {
  Object.defineProperty(source, key, {
    value,
    enumerable: false,
    writable: false,
    configurable: true,
    ...options,
  })

  return source
}

export function defineDefaults(source, defaults) {
  Object.entries(defaults).forEach(([key, value]) => {
    if (!source[key]) source[key] = value
  })

  return source
}
```

`writable: false` なので，`defineProperties` 側に渡されたものは書き換えられません．ライフサイクルメソッドは `defineDefaults` 側，`$` と `$$`（`COMPONENT_DOM_SELECTORS`）や `root` は `defineProperties` 側に入っています．だから `onMounted` は自分で書けば置き換わりますし，`this.root` は書き換えられません．

ついでに，`enumerable: false` が効いているところが後で出てきます．riot が足したプロパティは `Object.keys()` に出てこない，という話です．

`COMPONENT_LIFECYCLE_METHODS` の中身は，`src/core/component-lifecycle-methods.js` を見ると分かります．

```js
export const COMPONENT_LIFECYCLE_METHODS = Object.freeze({
  [SHOULD_UPDATE_KEY]: noop,
  [ON_BEFORE_MOUNT_KEY]: noop,
  [ON_MOUNTED_KEY]: noop,
  [ON_BEFORE_UPDATE_KEY]: noop,
  [ON_UPDATED_KEY]: noop,
  [ON_BEFORE_UNMOUNT_KEY]: noop,
  [ON_UNMOUNTED_KEY]: noop,
})
```

7 つとも `noop` です．だから `manage-component-lifecycle.js` の中は `if (this.onMounted)` のようなチェックをせずに，いきなり `this[ON_MOUNTED_KEY](...)` と呼べます．存在チェックを 1 箇所も書かないために，最初に全部埋めておく．これは素直に良いと思いました．

で，`curry` なのですが．

```js
curry(manageComponentLifecycle)(definedObject)
```

これは `(opts) => manageComponentLifecycle(definedObject, opts)` と同じで，アロー関数 1 行で書けます．`curri` という外部パッケージを引っ張ってきてまでやることではない気がします．こういうのが何箇所かあって，関数型のスタイルに寄せた結果，かえって 1 段読みにくくなっています．

なお，コンポーネントの CSS を `<style riot>` に流し込んでいるのもこのファイルの `cssManager.add(name, css)` の行です．ライフサイクルとは関係ない処理が 1 行だけ混ざっています．

## Object.create で共有される

`src/core/manage-component-lifecycle.js` の本体はこう始まります（3 つのメソッドの中身は後で見ます）．

```js
export function manageComponentLifecycle(
  component,
  { slots, attributes = [], props },
) {
  return autobindMethods(
    runPlugins(
      defineProperties(
        isObject(component) ? Object.create(component) : component,
        {
          mount(element, state = {}, parentScope) { /* ... */ },
          update(state = {}, parentScope) { /* ... */ },
          unmount(preserveRoot) { /* ... */ },
        },
      ),
    ),
    Object.keys(component).filter((prop) => isFunction(component[prop])),
  )
}
```

`Object.create(component)` がポイントで，クローンではなくプロトタイプチェーンを繋いでいます．`export default { ... }` のようにオブジェクトを書いたコンポーネントであれば，そのオブジェクトは登録時に 1 個しか作られないので，同じタグを 100 個マウントしてもユーザーが書いたメソッドの実体は 1 個のままです．インスタンス側に生えるのは `mount` / `update` / `unmount` だけです．

`autobindMethods` の第 2 引数が `Object.keys(component)` なのも，ここと関係しています．プロトタイプ側にあるユーザー定義のメソッドを列挙して，`this` を束縛したものをインスタンス側に置き直しています．これがあるから，`.riot` の中で `onclick={ this.handleClick }` と書いても `this` が外れません．先に触れた `enumerable: false` が効くのはここで，riot が `defineProperties` で足したものは `Object.keys()` に出てこないので，この列挙に混ざりません．

処理の順番は内側から，`Object.create` → `defineProperties` → `runPlugins` → `autobindMethods` です．プラグインが `autobindMethods` の前に走るので，プラグインが足したメソッドも `this` が束縛されます．順番が逆だったら束縛されません．気づきにくいけれど効いている並びだと思います．

## mount

```js
mount(element, state = {}, parentScope) {
  // any element mounted passing through this function can't be a pure component
  defineProperty(element, IS_PURE_SYMBOL, false)
  this[PARENT_KEY_SYMBOL] = parentScope

  defineProperty(
    this,
    PROPS_KEY,
    Object.freeze({
      ...computeInitialProps(element, props),
      ...generatePropsFromAttributes(attributes, parentScope),
    }),
  )

  this[STATE_KEY] = computeComponentState(this[STATE_KEY], state)
  this[TEMPLATE_KEY_SYMBOL] = this.template.createDOM(element).clone()
  // get the attribute names that don't belong to the props object
  // this will avoid recursive props rendering https://github.com/riot/riot/issues/2994
  this[ROOT_ATTRIBUTES_KEY_SYMBOL] = getRootComputedAttributeNames(
    this[TEMPLATE_KEY_SYMBOL],
  )

  // link this object to the DOM node
  bindDOMNodeToComponentInstance(element, this)
  // add eventually the 'is' attribute
  component.name && addCssHook(element, component.name)

  // define the root element
  defineProperty(this, ROOT_KEY, element)
  // define the slots array
  defineProperty(this, SLOTS_KEY, slots)

  // before mount lifecycle event
  this[ON_BEFORE_MOUNT_KEY](this[PROPS_KEY], this[STATE_KEY])
  // mount the template
  this[TEMPLATE_KEY_SYMBOL].mount(element, this, parentScope)
  this[ON_MOUNTED_KEY](this[PROPS_KEY], this[STATE_KEY])

  return this
}
```

props が `Object.freeze` されています．子コンポーネントの中から `this.props.foo = 1` と書いても通りません．一方で state はこうです．

```js
export function computeComponentState(oldState, newState) {
  return {
    ...oldState,
    ...callOrAssign(newState),
  }
}
```

浅くマージするだけで，凍結も監視もしていません．props は上から来るので触らせない，state は自分のものなので触っていい，という線の引き方です．React の props/state とだいたい同じ考え方ですが，riot は props 側を実際に凍結しているぶん厳しいですね．

もうひとつ，`this[TEMPLATE_KEY_SYMBOL]` に入るのはテンプレートのオブジェクトで，このファイルはその中身を知りません．`@riotjs/dom-bindings` の import が 1 行もないまま，`.createDOM()` `.clone()` `.mount()` だけを呼んでいます．前回書いた「本体が薄い」というのは，具体的にはこういう状態を指しています．

`ROOT_ATTRIBUTES_KEY_SYMBOL` のところは，コメントを読まないと何のためか分かりませんでした．ルート要素に式つきの属性（`class={ ... }` など）を書いたとき，その属性名だけを覚えておく仕組みです．[issue 2994](https://github.com/riot/riot/issues/2994) を開くと，`class` が二重に出てしまうバグの報告になっています．こういう「なぜこの 1 行があるのか」が issue にリンクされているので，コードだけ睨んで悩む必要がありません．

## update

```js
update(state = {}, parentScope) {
  if (parentScope) {
    this[PARENT_KEY_SYMBOL] = parentScope
  }

  // filter out the computed attributes from the root node
  const staticRootAttributes = Array.from(
    this[ROOT_KEY].attributes,
  ).filter(
    ({ name }) => !this[ROOT_ATTRIBUTES_KEY_SYMBOL].includes(name),
  )

  // evaluate the value of the static dom attributes
  const domNodeAttributes = DOMattributesToObject({
    attributes: staticRootAttributes,
  })

  // Avoid adding the riot "is" directives to the component props
  // eslint-disable-next-line no-unused-vars
  const { [IS_DIRECTIVE]: _, ...newProps } = {
    ...domNodeAttributes,
    ...generatePropsFromAttributes(attributes, this[PARENT_KEY_SYMBOL]),
  }
  if (this[SHOULD_UPDATE_KEY](newProps, this[PROPS_KEY]) === false) return

  defineProperty(
    this,
    PROPS_KEY,
    Object.freeze({
      // only root components will merge their initial props with the new ones
      // children components will just get them overridden see also https://github.com/riot/riot/issues/2978
      ...(parentScope ? null : this[PROPS_KEY]),
      ...newProps,
    }),
  )

  this[STATE_KEY] = computeComponentState(this[STATE_KEY], state)
  this[ON_BEFORE_UPDATE_KEY](this[PROPS_KEY], this[STATE_KEY])

  // avoiding recursive updates
  // see also https://github.com/riot/riot/issues/2895
  if (!this[IS_COMPONENT_UPDATING]) {
    this[IS_COMPONENT_UPDATING] = true
    this[TEMPLATE_KEY_SYMBOL].update(this, this[PARENT_KEY_SYMBOL])
  }

  this[ON_UPDATED_KEY](this[PROPS_KEY], this[STATE_KEY])
  this[IS_COMPONENT_UPDATING] = false

  return this
}
```

update のたびに DOM から属性を読み直して props を作り直しています．React のように仮想 DOM の差分から props を渡すのではなく，本物の DOM を毎回読みます．

`shouldUpdate` の比較が `=== false` なのは，明示的に `false` を返したときだけ更新を止める，という設計だからです．デフォルトは `noop` ですが，`@riotjs/util` の `noop` は `undefined` ではなく `this` を返します．

```js
// does simply nothing
export function noop() {
  return this
}
```

なのでデフォルトのままだとコンポーネント自身（truthy）が返ってきます．ここを `if (!result) return` にしてしまうと，`shouldUpdate` を書いたけれど `return` を書き忘れたコンポーネントが一切更新されなくなるので，`=== false` のほうが安全側ですね．

`(parentScope ? null : this[PROPS_KEY])` の行にもコメントがついています．`riot.mount()` で直接マウントしたルートコンポーネントは前の props とマージ，親から差し込まれた子コンポーネントは上書きです（[issue 2978](https://github.com/riot/riot/issues/2978)）．親が持っている値が正なので，子の古い props を残すと親と食い違う，ということだと思います．

`IS_COMPONENT_UPDATING` のフラグは [issue 2895](https://github.com/riot/riot/issues/2895) で，`onBeforeUpdate` の中で `this.update()` を呼ぶと無限ループする問題への対処です．フラグが立っているあいだはテンプレートの更新をスキップします．ライフサイクルフック自体は毎回呼ばれるので，ログを仕込んで動きを確かめると分かりやすいです．

なお，riot には値の変更を検知する仕組みがありません．`src/` を `Proxy` や getter/setter で grep しても 1 件も出てこないので，`this.state.count++` と書いただけでは画面は変わりません．`update()` を呼ぶまで何も起きません．これを不便と取るか，いつ再描画が走るか分かるほうが良いと取るかは好みだと思います．自分はどちらかというと後者で，`update()` の呼び出し箇所を grep すれば再描画のタイミングが全部出るのは楽でした．

## unmount の引数が 3 状態

最後の `unmount` に，読んでいて一番驚いた箇所があります．

```js
unmount(preserveRoot) {
  this[ON_BEFORE_UNMOUNT_KEY](this[PROPS_KEY], this[STATE_KEY])

  // make sure that computed root attributes get removed if the root is preserved
  // https://github.com/riot/riot/issues/3051
  if (preserveRoot)
    this[ROOT_ATTRIBUTES_KEY_SYMBOL].forEach((attribute) =>
      this[ROOT_KEY].removeAttribute(attribute),
    )
  // if the preserveRoot is null the template html will be left untouched
  // in that case the DOM cleanup will happen differently from a parent node
  this[TEMPLATE_KEY_SYMBOL].unmount(
    this,
    this[PARENT_KEY_SYMBOL],
    preserveRoot === null ? null : !preserveRoot,
  )
  this[ON_UNMOUNTED_KEY](this[PROPS_KEY], this[STATE_KEY])

  return this
}
```

引数は boolean のつもりの名前なのに，`null` に特別な意味があります．

- `true` なら，ルート要素を残して，式で生成した属性だけ消す
- `false` または未指定なら，ルート要素ごと消す
- `null` なら，HTML には一切触らない

3 つ目は，親がまとめて DOM を捨てる場面のためのものです．子が 1 個ずつ自分の DOM を消しに行くと無駄なので，親から `null` を渡してスキップさせます．

面白いのは，この引数の呼び名が 3 箇所で揺れていることです．

| 場所 | 名前 | 型 |
| --- | --- | --- |
| `src/core/manage-component-lifecycle.js` | `preserveRoot` | 記載なし |
| `src/api/unmount.js` の JSDoc | `keepRootElement` | `{boolean\|null}` |
| `riot.d.ts` | `keepRootElement` | `boolean \| undefined` |

JSDoc には `null` が書いてあるのに，`riot.d.ts` の `unmount(keepRootElement?: boolean)` には出てきません．内部用の値なので型では隠したいのだと思いますが，TypeScript から `null` を渡そうとすると怒られます．こういう，実装のほうが型より少し多いことをしている箇所は，読んでいると他にもちらほらあります．

## まとめ

174 行のうち，ほとんどが `mount` / `update` / `unmount` の 3 つに入っています．外部パッケージの import は `@riotjs/util` だけで，テンプレートエンジンのことは何も知りません．

このファイルを読み終わった時点で，riot がやっていないことがはっきりします．値の変更検知をしない，仮想 DOM を持たない，非同期のスケジューリングをしない．`update()` が呼ばれたらその場で同期的にテンプレートを更新して返ります．だから 15,856 バイトに収まっているわけです．

一方で，実際に DOM を書き換えているのは `this[TEMPLATE_KEY_SYMBOL].update()` の 1 行の向こう側で，そこはまだ何も読んでいません．

## 次回

リポジトリを移って `riot/dom-bindings` に入ります．`.createDOM()` `.clone()` `.mount()` `.update()` の中身です．たぶんここからのほうが面白いので，楽しみにしていてください 💁
