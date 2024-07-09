---
title: "Part5 データ取得用のサービスの作成"
---

[前回](/books/riotjs-tour-of-heroes/hero_detail.md) に引き続き，今回は Angular フレームワークに備わっている Service という機能に相当するコンポーネントと，メッセージを表示するコンポーネントを作成していきます．

では今回もやっていきましょう！

# サービスの追加

現在の実装では，heroes コンポーネント内で直接ヒーローデータを取得し保持していますが，コンポーネント内で直接データの取得や保存を行うべきではありません．コンポーネントはデータの受け渡しや表示に専念し，その他の処理は別ファイルへと切り出す方が望ましいです．

Angular チュートリアルの方針に沿って，アプリケーション全体でヒーローデータを取得できる `Hero` サービスを作成していきたいと思います．

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

これで，ヒーローデータ取得のロジックを `heroes` コンポーネントから切り出せました．ここから，Angular の ToH では非同期にデータをフェッチするために [RxJS](https://rxjs.dev/) というライブラリの `Observable` クラスを利用していきますが，今回は非同期処理については割愛します．

:::details riot で非同期処理の実装

一応実装してみたいという方向けに，少し原始的ですがやり方を以下に記載しておきます．

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

// グローバルに非同期関数をモジュールとして定義
export const myMethod = async () => {
  const response = await fetch(/** URL */)
  const data = response.json();
}
```
:::

以上で，非同期処理へのロジック変更は完了です．実際に動作させてみると，今まで実装したものと同じ挙動をすると思います．
