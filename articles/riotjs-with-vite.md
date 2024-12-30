---
title: 'Riot.js with Vite の実行環境試してみた'
emoji: '⚙️'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['riotjs', 'riot', 'vite', '開発環境']
published: true
---

:::message
本記事は [Riot.js Advent Calendar 2024](https://qiita.com/advent-calendar/2024/riotjs) の第 18 日目(遅刻組) の記事になります！
:::

https://qiita.com/advent-calendar/2024/riotjs

こんにちは．株式会社カミナシで EM をしている [Keeth](https://twitter.com/kuwahara_jsri) こと桑原です．

Web フロントエンドの開発環境におけるバンドラのデファクトスタンダードは [webpack](https://webpack.js.org/) でした．これは色んなところで数字にも現れておりますが，例として [State of JS 2024](https://2024.stateofjs.com/en-US/libraries/build_tools/) の結果から引用します．

![Build tools](/images/build_tools_ratios.png)

しかし，昨今の流行やここ数年の伸び率のトップは上の画像から明らかに [Vite](https://vite.dev/) であると思います．私も最近はプライベートで開発環境を作るときは `vite` と公式が用意している豊富なテンプレート^[公式リポジトリのテンプレート一覧は[こちら](https://github.com/vitejs/vite/tree/main/packages/create-vite)]から選ぶことがほとんどです．

しかし，テンプレート一覧には `React` や `Vue`, `Svelte`, `Qwik` など，有名なライブラリのものはあります（それはそう）が，[Riot.js（以下，riot）](https://riot.js.org/) に対応したものがありません．^[まぁ，ないなら作って PR 出せばいいじゃん？という話ではありますが，個別のフレームワーク対応の PR を全て対応することは難しく，riot は割と古参なライブラリでもあるため，おそらくマージされることは現実的ではないだろうなと思っています] とは言え，`Vanilla` もありますし，作ることは出来るだろうとは思っていたのでやってみたというのが今回のお話です 💁

## 先行事例

実は，海外のエンジニアの方で先行して試してみた方が，[riot の discord チャンネル](https://discord.gg/PagXe5Y) にリポジトリとブログを公開してくださっていました．しかも，チュートリアル形式で書かれておりまして，非常に参考になりました．こちらです 💁

https://dev.to/steeve/riotjs-vitejs-tutorial-fpn

参考になったと言うか，もうこれで良いんじゃないかと思いました（笑）vite の公式ドキュメントの [Rollup プラグインの互換性](https://ja.vite.dev/guide/api-plugin.html#rollup-plugin-compatibility) にもこのように書かれており，

> かなりの数の Rollup プラグインが Vite プラグインとして直接動作します（例: @rollup/plugin-alias や @rollup/plugin-json など）

riot にも [rollup-plugin-riot](https://www.npmjs.com/package/rollup-plugin-riot) があるのでワンちゃん行けるかな？と思いましたが，案の定動きました 🎉 また，riot の公式 CLI が用意している riot のボイラープレート^[`npm init riot` と叩くと対話的に環境が作れます]があり，こちらで作られる環境に沿ったほうが良いと考え，マージしたものを作って自分のリポジトリとして公開しました．

https://github.com/kkeeth/riotjs-with-vite

## 本格的に試してみる

以前，riot のチュートリアル本として [Riot.js で Tour of Heroes を試す](https://zenn.dev/kkeeth/books/riotjs-tour-of-heroes) という無料デジタル本を zenn にて公開しました 💁^[riot には公式のチュートリアルがないため，自分としても欲しかったので書きました．]この本の `Chapter 8` が `発展② Vite を用いた開発環境` というタイトルになっておりますが，未だに書けていないので，先に試してみます(ｵｲ

### Goodbye webpack

まずはベースバンドラにしている `webpack` を削除します．

- 不要になるモジュールを削除

  ```bash
  $ pnpm rm webpack webpack-cli webpack-dev-server html-webpack-plugin mini-css-extract-plugin css-loader @riotjs/webpack-loader
  ```

- 不要になった設定ファイル（`webpack.config.js`）を削除

### Hello vite!

続いて，`vite` をインストールします．

- `vite` と `rollup-plugin-riot` のインストール

  ```bash
  $ pnpm i -D vite rollup-plugin-riot
  ```

- `package.json` の `scripts` の書き換え

  ```diff
  +    "dev": "vite",
  +    "build": "vite build",
  +    "preview": "vite preview",
       "test": "NODE_OPTIONS=\"--loader @riotjs/register\" c8 mocha --require jsdom-global/register src/**/*.spec.js",
       "cov": "c8 report --reporter=text-lcov",
       "cov-html": "c8 report --reporter=html",
  -    "build": "webpack --mode production",
  -    "prepublishOnly": "npm test",
  -    "start": "webpack serve --mode development --hot --port 3000"
  +    "prepublishOnly": "npm test"
  ```

- 設定ファイル（`vite.config.js`）を作成し，以下を追記

  ```js:vite.config.js
  import { defineConfig } from "vite";
  import riot from "rollup-plugin-riot";
  import path from 'node:path';

  export default defineConfig({
    root: process.cwd(),
    plugins: [riot()],
    resolve: {
      alias: {
        '@': path.resolve(__dirname, './src'),
        '@components': path.resolve(__dirname, 'src/components'),
        '@services': path.resolve(__dirname, 'src/services'),
      },
    },
    build: {
      outDir:
        "dist" /** https://vitejs.dev/config/build-options.html#build-outdir */,
      minify:
        "esbuild" /** https://vitejs.dev/config/build-options.html#build-minify */,
      target:
        "esnext" /** https://vitejs.dev/config/build-options.html#build-target */,
    },
  });
  ```

### 各種設定とコードの見直し

- `index.html` にメインとなる `index.js` を読み込むように変更

  ```diff
    <div class="container">
      <div id="root"></div>
  +   <script type="module" src="/index.js"></script>
    </div>
  ```

- `index.html` と `index.js` をドキュメントルートに移動

  ```
  src/index.html -> index.html
  src/index.js -> index.js
  ```

- `src/register-global-components.js` を修正

  ```diff
  -const globalComponentsContext = import.meta.webpackContext(
  -  "./components/",
  -  { recursive: true, regExp: /[a-zA-Z0-9-]+\.riot/ },
  +const globalComponentsContext = import.meta.glob(
  +  "./components/**/*.riot",
  +  { eager: true },
   );

   export default () => {
  -  globalComponentsContext.keys().map((path) => {
  +  Object.entries(globalComponentsContext).map(([path, component]) => {
       const name = basename(path, ".riot");
  -
  -    const component = globalComponentsContext(path);

       register(name, component.default || component);
  ```

以上で，今まで通り Tour of Heroes のデモが動くようになりました！完成版のリポジトリのリンクも共有します 💁

https://github.com/kkeeth/riot-toh-demo/tree/with-vite

## 終わりに

やってみたらやはりサクッと終わってしまったので，さっさとやっておけばよかったなぁという反省でした（笑）それにしても vite はとても優秀でありがたいなとつくづく感じますし，何より起動とホットリロードが速い！

次は msw を用いたテストも書いていきたいところです．というのも，riot 公式のテストでは，JavaScript のテスティングフレームワークとしては最古参と言っても良い [mocha](https://mochajs.org/) をベースに環境が用意されていますので，せっかく vite を用いているのであれば [vitest](https://vitest.dev/) を試してみたいところです．^[全然動かなかったらどうしよう…w]

ではでは(=ﾟ ω ﾟ)ﾉ
