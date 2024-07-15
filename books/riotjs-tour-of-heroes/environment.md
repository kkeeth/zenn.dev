---
title: "Chapter1 新規プロジェクトの作成"
---

[Riot.js](https://riot.js.org/)（以下，riot）は非常にシンプルかつ軽量で入門の敷居も低く，とても書きやすいコンポーネント指向の UI ライブラリです．

本シリーズは，基本的に [Angular 公式のチュートリアル「Tour of Hero」](https://angular.jp/tutorial) を riot でなぞっていく本になります．少々長いため，ここから何回かに分けてシリーズものとして書いていこうと思っていますので，riot 入門者のチュートリアルとして気長に見ていただければ幸いです 🙇

それではやっていきましょう！

# プロジェクトの雛形を作成

まずは本プロジェクトのディレクトリを作成して移動し，riot 公式のボイラープレートを用いて雛形を作成します．今回は `webpack` を使うことにします．では，以下のコマンドをターミナルから実行してください．

以下，適宜ご自身のものに置き換えてください，特に指定がなければ `Enter` を押下してください．

```bash
# ディレクトリ作成および移動
$ mkdir riot-tour-of-heroes && cd riot-tour-of-heroes

# riot プロジェクトの作成
$ npm init riot

# いつもの npm init の手順ですので，お好きに設定してください
package name: (riot-tour-of-heroes)
version: (1.0.0)
description:
entry point: (index.js)
test command:
git repository: (ssh://git@github.com/kkeeth/riot-tour-of-heroes.git)
keywords: riot, riotjs, tour-of-heroes
author: kkeeth
license: (ISC) MIT
About to write to /Users/k-kuwahara/src/github.com/kkeeth/riot-tour-of-heroes/package.json:

...(省略)

Is this OK? (yes) y

# ここからが riot プロジェクトの設定
# 前述通り，今回は Webpack 版を選択
? Please select a template …
❯ Webpack Project Template
  Parcel Project Template
  Rollup Project Template
  Simple Component
  SPA (Webpack) Project Template
  Custom Template (You will need to provide a template path to your template zip file)

✔ Please select a template · webpack
✔ Downloading the template files
✔ Unzipping the file downloaded
✔ Deleting the zip file
✔ Copying the template files into your project
✔ Deleting the temporary folder
✔ Template successfully created!
```

以上でプロジェクトの作成は完了です！

なお，手前味噌ですが [create-riot についての記事](https://qiita.com/clown0082/items/9c908309c2031f398baf)も Qiita に書いておりますので，ご参考ください．

:::message
↑ の記事でも書いておりますが，執筆時点での最新の `create-riot` を用いたボイラープレートでは，`package.json` が圧縮（minify）されてしまいますので，`Prettier` や[外部サービス](https://one-ap-engineer.com/tools/json-formatter/) など何かしらのフォーマッターで整形することをお勧めします．
:::

以上で雛形の作成は完了です.


# アプリケーションを起動

プロジェクトの新規作成が終わりましたので，アプリケーションを起動してみたいと思いますが，その前に必要なモジュールをインストールする必要があります．以下のコマンドを実行してください.

```bash
$ npm install

# pnpm をご利用の方
$ pnpm install
```

※これ以降筆者は `npm` コマンドで統一していきますので，`pnpm` 等，他のものをご利用の方は適宜読み替えてください．`lock` ファイルも同様です．

インストールが完了しましたら，いよいよ起動してみます．以下のコマンドを実行してください.

```bash
$ npm run start
```

実行しますと，自動でアプリケーションが起動しブラウザも一緒に起動，[http://localhost:3000/](http://localhost:3000/) の画面が開くかと思います！

![アプリケーション起動](https://storage.googleapis.com/zenn-user-upload/wobh7fily17yhjpbzg6sny1rflcj)

# アプリケーションの変更

ここからは実際に開発に入る前の諸々の調整をしていきたいと思います．

## タイトルの変更

まずはタイトルを変更します．`index.html` を開き以下のように変更してください．

```diff
 <!DOCTYPE html>
 <html>
   <head>
-    <title>My Riot App</title>
+    <title>Tour of Heroes</title>
     <link
       rel="stylesheet"
```

更新後，ブラウザのタブのタイトルが変わっていれば OK です 👌

## スタイリングの初期化（+ CSS リセット）

次にデフォルトで指定されているスタイリングを初期化します．まずは `index.html` で読み込んでいるフォントと２つのスタイルシートを削除します．

```diff
-    <link
-      rel="stylesheet"
-      href="https://fonts.googleapis.com/css?family=Roboto:300,300italic,700,700italic"
-    />
-    <link
-      rel="stylesheet"
-      href="https://cdnjs.cloudflare.com/ajax/libs/normalize/8.0.1/normalize.css"
-    />
-    <link
-      rel="stylesheet"
-      href="https://cdnjs.cloudflare.com/ajax/libs/milligram/1.4.0/milligram.css"
-    />
   </head>
```

また，一般的には何かしらの CSS リセットのライブラリをインストールしますが，今回はなるべく Angular 公式のチュートリアルに従うため，割愛します．

ここまでできますと，スタイリングがあたっていない画面が表示されると思います．

![CSS 初期化後](https://storage.googleapis.com/zenn-user-upload/d93e611e9048-20240713.png)

以上でスタイリングの初期化は完了です．

:::details ress を用いた CSS リセット

もし riot で　CSS リセットライブラリを使う場合の方法を以下に記載しておきます．今回は [ress](https://github.com/filipelinhares/ress) を利用します．

```bash
$ npm install -D ress
```

先程インストールした `ress` をアプリケーション内で読み込んで行きます．`index.js` に以下を追記してください．

```diff
+ import "ress";
  import "./style.css";
  import "@riotjs/hot-reload";
  import { mount } from "riot";
  import registerGlobalComponents from "./register-global-components.js";
```

この辺は他の js ライブラリ・フレームワークと同じですね．
:::

## アプリケーションのベーススタイルを追加

では次にアプリケーション全体のベーススタイリングを設定してきます．`src` ディレクトリ直下の `style.css` というファイルに以下を追記してください．

```diff
- .container {
-   display: flex;
-   justify-content: center;
-   align-items: center;
-   min-height: 100vh;
- }
+ * {
+   font-family: Arial, Helvetica, sans-serif;
+ }
+ h1 {
+   color: #264D73;
+   font-size: 2.5rem;
+ }
+ h2, h3 {
+   color: #444;
+   font-weight: lighter;
+ }
+ h3 {
+   font-size: 1.3rem;
+ }
+ body {
+   padding: .5rem;
+   max-width: 1000px;
+   margin: auto;
+ }
+ @media (min-width: 600px) {
+   body {
+     padding: 2rem;
+   }
+ }
+ body, input[text] {
+   color: #333;
+   font-family: Cambria, Georgia, serif;
+ }
+ a {
+   cursor: pointer;
+ }
+ button {
+   background-color: #eee;
+   border: none;
+   border-radius: 4px;
+   cursor: pointer;
+   color: black;
+   font-size: 1.2rem;
+   padding: 1rem;
+   margin-right: 1rem;
+   margin-bottom: 1rem;
+   margin-top: 1rem;
+ }
+ button:hover {
+   background-color: black;
+   color: white;
+ }
+ button:disabled {
+   background-color: #eee;
+   color: #aaa;
+   cursor: auto;
+ }
+ hr {
+   margin-top: 3rem;
+ }
```

ここまで書けましたら，以下の画像のようにスタイリングが変更されていると思います．

![ベーススタイル設定後](/images/books/riotjs_toh/01_completed.png)


# `webpack` のエイリアスを設定

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

これにより，今後は

```js
import hoge from "@/components/hoge.riot";
```

のように書くことができますし，この書籍でも利用していきますので，是非設定してください．以上で Chapter1「新規プロジェクトの作成」は完了です！
