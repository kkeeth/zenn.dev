---
title: 'Riot.js 10 のソースコードを読む (1) どこに何があるか'
emoji: '🗺️'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['riotjs', 'riot', 'javascript', 'ソースコードリーディング']
published: false
---

こんにちは．[Keeth](https://x.com/kuwahara_jsri) こと桑原です．

長らく [Riot.js（以下，riot）](https://riot.js.org/) を使ってきたのに，中身をちゃんと読んだことがありませんでした．riot は 2013 年に Tero Piirainen 氏が始めたライブラリで，いまバージョン 10 まで来ています．メジャーバージョンは 10 まで上がっているのに，コアの考え方はほとんど変わっていません．React が hooks で書き方を作り変え，Vue が Composition API を足していった十年のあいだ，riot はだいたい同じところに立っています．

その割に，中身を読んだという話をあまり見かけません．というわけで，読みながら書いていくシリーズを始めます．先に全部分かっている人間の解説ではないので，詰まったところは詰まったまま書きます．間違っている記述があればバンバンコメントいただけると嬉しいです 🙇‍♂️

:::message
本記事の数字やコードは，`riot/riot` の **v10.1.6**（本記事執筆時点の最新）を対象にしています．
:::

## TL;DR

- riot は `riot` / `dom-bindings` / `compiler` / `util` の 4 リポジトリに分かれていて，`riot/riot` の `src/` は 1081 行しかない
- 定数と Symbol は全部 `@riotjs/util` にあるので，先に `npm install` して `node_modules/@riotjs/util/` を開けるようにしておくと楽
- `src/core/manage-component-lifecycle.js` の 174 行に `mount` / `update` / `unmount` の実体が全部入っている．ここから読むのがおすすめ
- リポジトリのルートにある `riot.js` や `riot.min.js` はコミットされたビルド成果物なので，読むのは `src/` 以下だけでよい
- `test/components/` のファイル名と `src/` のコメントに GitHub issue 番号が入っていて，「なぜこの分岐があるのか」を追える

## まず量を把握する

読む前に，どれくらいの量なのかを数えました．

```bash
$ find src -name '*.js' | wc -l
      45
$ find src -name '*.js' -exec cat {} + | wc -l
    1081
```

| 対象 | 量 |
| --- | --- |
| `src/` | 45 ファイル，1081 行 |
| `test/specs/` | 8 ファイル，1590 行 |
| `riot.min.js` | 15,856 バイト |
| `dependencies` | 1 個（`@riotjs/dom-bindings`） |

テストのほうが本体より 509 行多いです．そして 1081 行というのは，週末に読み切れる量です．これなら行けそうだと思って手を付けました．

## 最初の落とし穴

`riot/riot` を clone して `src/` を開くと，こういう import が並んでいます．`src/core/manage-component-lifecycle.js` の冒頭からの抜粋です．

```js
import {
  IS_COMPONENT_UPDATING,
  IS_PURE_SYMBOL,
  ON_BEFORE_MOUNT_KEY,
  // ...
  PARENT_KEY_SYMBOL,
  PROPS_KEY,
  ROOT_KEY,
  STATE_KEY,
  TEMPLATE_KEY_SYMBOL,
  autobindMethods,
  defineProperties,
  // ...
} from '@riotjs/util'
```

`PROPS_KEY` が何なのか，`PARENT_KEY_SYMBOL` が文字列なのか Symbol なのかが，このリポジトリでは分からず，全て `@riotjs/util` にあります．同じように，テンプレートを実際に DOM へ反映する仕事は `@riotjs/dom-bindings` に，`.riot` ファイルを JavaScript に変換する仕事は `@riotjs/compiler` にあります．

つまり riot は 4 つのリポジトリに割れています．

| リポジトリ | 担当 |
| --- | --- |
| [riot/riot](https://github.com/riot/riot) | コンポーネントのライフサイクル管理（1081 行） |
| [riot/dom-bindings](https://github.com/riot/dom-bindings) | 式の評価と DOM 更新 |
| [riot/compiler](https://github.com/riot/compiler) | `.riot` → JavaScript の変換 |
| [riot/util](https://github.com/riot/util) | 定数や Symbol と，共有のグローバル状態 |

`riot/riot` が薄いのは，薄くしたからではなく，仕事を 3 つ外に出しているからです．これを知らずに読み始めると，「本体を全部読んだのに何も分かっていない」という状態になります．自分は一度なりました．

なので先に `npm install` して `node_modules/@riotjs/util/` を開けるようにしておくことをおすすめします．定数の正体が分からないまま `src/core/` を読むのはなかなかきついです．ちなみに `@riotjs/util` は `dependencies` ではなく `devDependencies` に入っていて，配布ファイルにはビルド時にバンドルされて埋め込まれています．

## riot/riot が担当していること

外に出した結果，本体に残っているのはこれだけになります．

- `.riot` をコンパイルした結果のオブジェクトを受け取って，マウント可能な関数に変える
- そのオブジェクトに `mount` / `update` / `unmount` とライフサイクルメソッドを生やす
- 名前とコンポーネントの対応表を持つ
- CSS を `<style riot>` に流し込む

テンプレートの解析もしませんし，式の評価もしません．`src/` の中で `@riotjs/dom-bindings` を import しているファイルを数えると，45 個中 6 個だけです．

```bash
$ grep -rln "@riotjs/dom-bindings" src
src/api/__.js
src/core/component-template-factory.js
src/core/create-attribute-bindings.js
src/core/create-component-from-wrapper.js
src/utils/create-runtime-slots.js
src/utils/get-root-computed-attribute-names.js
```

残り 39 個は，テンプレートエンジンの存在を知らずに書かれています．

## いきなり読むべき 1 ファイル

先に結論を書きます．**`src/core/manage-component-lifecycle.js` の 1 ファイル，174 行を読めば riot の 9 割は分かります**．

`mount()` も `update()` も `unmount()` も，実体は全部ここにあります．props の凍結も，親スコープの保持も，再帰更新の防止も，ライフサイクルフックの呼び出しもここです．他の 44 ファイルは，このファイルに渡すオブジェクトを組み立てるか，このファイルが呼ぶ小さな関数か，どちらかでしかありません．

これは言い切れる主張なので，読んで違うと思ったら反論してください．自分は `src/api/` から順番に読んで 2 時間くらい迷子になったあと，このファイルに着いて「ここが全部だったのか」となりました．先に言っておけば省ける時間だと思います．

読む順番としてはこうなります．

1. `riot.d.ts` の `interface RiotComponent`．出来上がりの形が全部書いてあります
2. `src/riot.js`．12 行．公開 API の一覧です
3. `src/core/manage-component-lifecycle.js`．本体です
4. `src/core/create-component-from-wrapper.js` と `src/core/instantiate-component.js`．3 に渡すものを作っています
5. 残り

`riot.d.ts` を最初に見ると効率が良いです．`src/` がやっているのは結局のところ，この型のオブジェクトを 1 個作ることだからです．ゴールの形を知ってからコードに入ると，どの処理がどのプロパティを埋めているのかという観点で読めます．

ちなみに 2 の `src/riot.js` は，本当に `export * from` が 12 行並んでいるだけのファイルです．

```js
export * from './api/register.js'
export * from './api/unregister.js'
export * from './api/mount.js'
export * from './api/unmount.js'
export * from './api/install.js'
export * from './api/uninstall.js'
export * from './api/component.js'
export * from './api/pure.js'
export * from './api/create-pure-component.js'
export * from './api/with-types.js'
export * from './api/version.js'
export * from './api/__.js'
```

## 読まなくていいもの

リポジトリのルートに `riot.js` と `riot.min.js`，そして `riot+compiler.js` と `riot+compiler.min.js` があります．これは Makefile の `build` ターゲットが生成したビルド成果物がそのままコミットされているもので，ソースではありません．`riot.js` を開いて読み始めると，`@riotjs/util` や `@riotjs/dom-bindings` まで含めてインライン展開されたものを読むことになるので，時間を捨てます．ソースは `src/` の下だけです．

`esm/` や `cjs/` は `.gitignore` に入っているので，clone した直後には存在しません．npm のパッケージには含まれます．

あと `src/core/create-attribute-bindings.js` は，`src/` からも `test/` からも `riot.d.ts` からも一度も参照されていません．

```bash
$ grep -rn "createAttributeBindings" src test riot.d.ts
src/core/create-attribute-bindings.js:10:export function createAttributeBindings(node, attributes = []) {
```

自分の定義しか出てきません．22 行のデッドコードなので飛ばしていいと思います．こういうのが残っているのはまあ，ありますね．

## テストが実質のドキュメント

`test/components/` に 69 個の `.riot` ファイルがあって，そのうち 14 個がこういう名前をしています．

```
issue-2895-parent.riot
issue-2978-child.riot
issue-2994-class-duplication.riot
issue-3051-class-duplication.riot
issue-3055-overwrite.riot
```

GitHub の issue 番号がそのままファイル名になっています．そして `src/core/manage-component-lifecycle.js` の中にも，同じ番号が URL つきでコメントされています．

```js
// avoiding recursive updates
// see also https://github.com/riot/riot/issues/2895
if (!this[IS_COMPONENT_UPDATING]) {
```

なぜこの分岐があるのかが，issue を開けば分かるようになっています．コードだけ読んで「何のための if だろう」と悩む時間が要りません．公式ドキュメントより，このコメントと issue のほうが情報量が多い場面がけっこうあります．

## 次回

`riot.register()` と `riot.mount()` を繋いでいるものを見ます．この 2 つの関数のあいだに import の関係が一切なくて，最初に読んだとき素直に分からなかった箇所なので，そこから始めます．

https://zenn.dev/keeth/articles/riotjs10-source-reading-02-register-mount
