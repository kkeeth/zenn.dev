---
title: "Part5 データ取得用のサービスの作成"
---

[前回](/books/riotjs-tour-of-heroes/hero_detail.md) に引き続き，今回は Angular フレームワークに備わっている Service という機能に相当するコンポーネントと，メッセージを表示するコンポーネントを作成していきます．

では今回もやっていきましょう！

# `Hero` サービスの追加

現在の実装では，`heroes` コンポーネント内で直接ヒーローデータを取得し保持していますが，コンポーネント内で直接データの取得や保存を行うべきではありません．コンポーネントはデータの受け渡しや表示に専念し，その他の処理は別ファイルへと切り出す方が望ましいです．

`Angular` チュートリアルの方針に沿って，アプリケーション全体でヒーローデータを取得できる `Hero` サービスを作成していきたいと思います．

## `Hero` サービスの作成

ということで，まずは以下のディレクトリとファイルを作成してきます．

- `src/services` ディレクトリ
- `src/services/hero.service.js` ファイル

ファイルができましたら中身を書いていきましょう．

## ヒーローデータの取得

では，現在 `heroes` コンポーネントで取得しているヒーローデータを `hero.service` で取得するように変更していきます．

まず `hero.service.js` ファイルにて `HEROES` 配列を読み込み，コール元にレスポンスするメソッド `getHeroes()` を定義します．

```js
import { HEROES } from "@/components/heroes/mock";

export const getHeroes = () => {
  return HEROES;
};
```

:::details @ エイリアス
ファイルの `import` を相対パスで記述することも可能ですが，ドキュメントルートや今回のように `src` フォルダをベースとしてパスを指定したい，というオーダーもあると思います．

その場合，webpack の `alias` という API でいけますが，riot のボイラープレートは `"type": "module"` のため，ちょっとテクニカルですが，以下のようなハックをすると可能です．

```diff
  import MiniCssExtractPlugin from "mini-css-extract-plugin";
  import webpack from "webpack";
  import path from "node:path";
+ import { fileURLToPath } from 'url';
+ const __filename = fileURLToPath(import.meta.url);
+ const __dirname = path.dirname(__filename);

（中略）

+  resolve: {
+    alias: {
+      '@': path.resolve(__dirname, 'src')
+    }
+  },
   module: {
     rules: [
       {
```
:::

次に，`heroes.riot` にて先程定義したメソッドをマウント前に処理する `onBeforeMount()` メソッドでコールし，`heroes` 変数の初期値としてセットします．

```diff

  <script>
-   import { HEROES } from "../mock-heroes";
+   import { getHeroes } from '@/services/hero.service';
    import HeroDetail from '../hero-detail/hero-detail.riot';

    export default {
      // 厳密にはこの記述はなくても問題ない
-     heroes: HEROES,
+     heroes: [],
      selectedHero: {},
+     onBeforeMount() {
+       this.heroes = getHeroes();
+     },
      handleInput(e) {
```

これで，ヒーローデータ取得のロジックを `heroes` コンポーネントから切り出せました．ただ，これですと現在はモックデータということもありますが，同期的にデータを取得しています．しかし実際のアプリケーションでは，外部 API をコールする事が多いでしょう．その場合，ブラウザがブロックされてしまわないように非同期処理を実装する必要があります．

ここから，`Angular` の ToH では非同期にデータをフェッチするために [RxJS](https://rxjs.dev/) というライブラリの `Observable` クラスを利用していきますが，riot では [riot/observable] を利用しつつ，非同期処理を実装していきます．

:::message
1️⃣ version 3 以前を使われている方は，`observable` はコアモジュールのため，以下のように本体からインポートが可能です．

```js
import { observable } from "riot"
```

2️⃣ `riot/observable` はイベントの送受信用のライブラリであり，非同期処理を内包しているわけではないため，非同期処理は別途書く必要があります．
:::

:::details riotコンポーネント内の非同期処理

riot コンポーネント内の組み込みメソッドでも，非同期処理を書くことはできます．以下例．

```js
export default {
  // レンダリング前に必ず実行
  async onBeforeMount() {
    const response = await fetch(/** URL */)
    const data = response.json();
  },
  // 非同期の関数を定義しておき，適宜呼び出したいところで呼び出す
  async myMethod(v) {
    const response = await fetch(/** URL */)
    const data = response.json();
  }
}
```
:::

まずは `riot/observable` をインストールしましょう．

```bash
$ npm install -S  @riotjs/observable
```

続いて，`riot/observable` を用いて `hero.service.js` を書き直していきたいのですが，

```diff
  import { HEROES } from "@/components/global/heroes/mock-heroes";
+ import observable from '@riotjs/observable'

export const getHeroes = () => {
  return HEROES;
};

const heroService = createObservable({
  heroes: [],

  async fetchHeroes() {
    try {
      const response = await fetch('https://api.example.com/heroes');
      const heroes = await response.json();
      this.heroes = heroes;
      this.trigger('heroesUpdated', this.heroes);
    } catch (error) {
      console.error('Failed to fetch heroes:', error);
    }
  },

  getHero(id) {
    return this.heroes.find(hero => hero.id === id);
  },

  addHero(hero) {
    this.heroes.push(hero);
    this.trigger('heroesUpdated', this.heroes);
  },

  deleteHero(id) {
    this.heroes = this.heroes.filter(hero => hero.id !== id);
    this.trigger('heroesUpdated', this.heroes);
  }
});

export default heroService;
```
# `Messages` サービスの追加

続いて，メッセージを表示するためのコンポーネントとサービスを実装していきます．まずは必要なフォルダとファイルを作ります．

* `src/components/global/messages/`
* `src/components/global/messages/messages.riot`
* `src/components/global/messages/messages.spec.js`
* `src/services/messages.service.js`

作成できましたら，`.spec.js` ファイルは後回しにして，`.riot` ファイルの方に以下を追記してください．中身は仮です．

```html
<messages>
  <p>messages</p>
</messages>
```

そうしましたら，これを読み込んで画面に表示させましょう．

```

以上で，非同期処理へのロジック変更は完了です．実際に動作させてみると，今まで実装したものと同じ挙動をすると思います．
