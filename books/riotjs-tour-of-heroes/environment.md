---
title: 'Part1 新規プロジェクトの作成'
---

[Riot.js](https://riot.js.org/)（以下，riot）は非常にシンプルかつ軽量で入門の敷居も低く，とても書きやすいコンポーネント指向の UI ライブラリです．

本シリーズは，基本的に [Angular 公式サイトのチュートリアルページ](https://angular.jp/tutorial) を riot でなぞっていく記事になります．少々長いため，ここから何回かに分けてシリーズものとして書いていこうと思っていますので，riot 入門者のチュートリアルとして気長に見ていただければ幸いです 🙇

それではやっていきましょう！

# プロジェクトの雛形を作成

まずは本プロジェクトのディレクトリを作成して移動し，riot 公式のボイラープレートを用いて雛形を作成します．今回は `webpack` を使うことにします．では，以下のコマンドをターミナルから実行してください．

※コードを GitHub で管理したい方は先にリポジトリを作成しておいてください 🙇

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
❯  Webpack Project Template
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

※ ↑ の記事でも書いておりますが，執筆時点での最新の `create-riot` を用いたボイラープレートでは，`package.json` が圧縮（minify）されてしまいますので，`Prettier` や[外部サービス](https://one-ap-engineer.com/tools/json-formatter/) など何かしらのフォーマッターで整形することをお勧めします．
以上で雛形の作成は完了です.

# アプリケーションを起動

プロジェクトの新規作成が終わりましたので，アプリケーションを起動してみたいと思いますが，その前に必要なモジュールをインストールする必要があります．以下のコマンドを実行してください.

```bash
$ npm install

# yarn をご利用の方
$ yarn
```

※これ以降筆者は `npm` コマンドで統一していきますので，`yarn` をご利用の方は適宜読み替えてください．

インストールが完了しましたら，いよいよ起動してみます．以下のコマンドを実行してください.

```bash
$ yarn start
```

実行しますと，自動でアプリケーションが起動しブラウザも一緒に起動，[http://localhost:3000/](http://localhost:3000/) の画面が開くかと思います！

![アプリケーション起動](https://storage.googleapis.com/zenn-user-upload/wobh7fily17yhjpbzg6sny1rflcj)

ちなみにですが，この時点で `package.json` にいくつかのコマンドが自動で追記されておりますのでご確認ください．

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

## スタイリングの初期化（CSS リセット）

次にデフォルトで指定されているスタイリングを初期化します．まずは `index.html` で指定されている２つのスタイルシートを削除します．

```diff
       rel="stylesheet"
       href="https://fonts.googleapis.com/css?family=Roboto:300,300italic,700,700italic"
     />
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

完了しましたら，次に CSS リセットのライブラリをインストールします．今回は [ress](https://github.com/filipelinhares/ress) を利用します．

```bash
$ npm install --save-dev ress
```

では最後に，先程インストールした `ress` をアプリケーション内で読み込んで行きます．`index.js` に以下を追記してください．

```diff
 import "@riotjs/hot-reload";
 import { component } from "riot";
 import App from "./app.riot";
 import registerGlobalComponents from "./register-global-components";
+import "ress";
```

ここまでできますと，以下の画像のようにスタイリングがあたっていない画面が表示されると思います．

![リセットCSS適用後](https://storage.googleapis.com/zenn-user-upload/xassl1mdrjsijd4nkkacw4hwzm8s)

以上でスタイリングの初期化は完了です．

## アプリケーションのベーススタイルを追加

では次にアプリケーション全体のベーススタイリングを設定してきます．`src` ディレクトリ直下に `style.css` というファイルを作成し以下の内容追記してください．

```css
/* Application-wide Styles */
h1 {
  color: #369;
  font-family: Arial, Helvetica, sans-serif;
  font-size: 250%;
}
h2,
h3 {
  color: #444;
  font-family: Arial, Helvetica, sans-serif;
  font-weight: lighter;
}
body {
  margin: 2em;
}
body,
input[type='text'],
button {
  color: #333;
  font-family: Cambria, Georgia;
}
/* everywhere else */
* {
  font-family: Arial, Helvetica, sans-serif;
}
```

ここまでできますと，以下の画像のようにスタイリングが変更されていると思います．以上で Part1「新規プロジェクトの作成」は完了です！

![ベーススタイル設定後](https://storage.googleapis.com/zenn-user-upload/g1lnhj1g0yupzpzdcgqr1fvtsrfh)

では，[Part2「ヒーローエディタ」](/books/riotjs-tour-of-heroes/2)に続きます．
