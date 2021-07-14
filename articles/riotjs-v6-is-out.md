---
title: 'Riot.js v6 is out🎉 ※後ほど更新します'
emoji: '✊'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['riotjs', 'JavaScript', 'release']
published: true
---

# 🚧 2021-07-14 記事内ではまだアプリが動いていないため，対応後更新します 🚧

https://twitter.com/riotjs_/status/1414206624496037888

という事で，[Riot.js（以下，riot）](https://riot.js.org) のバージョン v6.0.0 がリリースされました 🎉 ツイートにもあるように，目玉の更新は `TypeScript` の対応が進んだ形ですね！これは嬉しい．まだ公式ドキュメントの更新はされていないようですので，更新され次第日本語訳も対応します．

# リリースノート

公式の[リリースノート](https://github.com/riot/riot/releases/tag/v6.0.0) を見てみますと，２件のバグ修正と残りは `TypeScript` の対応のみでしたね．また，

> Add strict typescript support [#2912](https://github.com/riot/riot/pull/2912) **Breaking Change: the Riot.js types have been updated**

> If you are not a typescript user this release doesn't introduce any braking change.

とあるように，今回の更新にも**破壊的変更が含まれており**，riot を更新して `TypeScript` を導入する場合は riot の型が更新されています．

今までは手動で `TypeScript` を導入（`*.d.ts` 含む）しなければならず，一応公式のサンプルリポジトリにも typescript の例は作成されていましたので，こちらをコピーして利用するかぐらいでしたが，今度からはデフォルトで使えるのでこれはありがたいです．

https://github.com/riot/examples/tree/gh-pages/typescript

# v6 での `TypeScript` の導入の仕方

まずはリリースノートに記載のあったサンプルコードを見てみましょう．

```html
<my-component>
  <p>{ state.greeting }{ props.username }</p>

  <!--
    Notice that lang="ts" does actually nothing.
    It's just needed to let your IDE interpret the below code as typescript.
    The riot compiler will ignore it
  -->
  <script lang="ts">
    import { withTypes, RiotComponent } from 'riot'

    export type MyComponentProps = {
      username: string
    }

    export type MyComponentState = {
      message: string
    }

    export type MyComponent = RiotComponent<MyComponentProps, MyComponentState>

    export default withTypes<MyComponent>({
      state: {
        message: 'hello',
      },
    })
  </script>
</my-component>
```

なるほど，眺めるだけでだいたい riot に `TypeScript` に導入の仕方は理解しました．

- `<script>` タグに `lang="ts"` を指定
- riot 本体から `withTypes` をインポートする
- props の型を定義
- state の型を定義
- 👆 ２つの型定義を用いて riot コンポーネントの型を定義
- ラストに 👆 で定義した riot コンポーネントの型定義と `withTypes` メソッドでインスタンスを作成しエクスポート

また，本体のソースコードの package.json を見ましたら，しっかりと `typescript: ^4.3.5` と記載がありましたね．

https://github.com/riot/riot/blob/main/package.json

少々余談をすると，[v3 のプリプロセッサの指定](https://v3.riotjs.vercel.app/guide/#pre-processor)の仕方が `<script>` タグに `type="ts"` でしたね．v6 では `lang="ts"` と指定するようになりましたが，指定の仕方の復活した感じで懐かしみを覚えました 😆

# もう少し実践的に確認

とりあえず，公式が用意しているプロジェクト作成用のコマンドから riot のバージョンを v6 に上げてみて動くか確認してみます．という事で以下のコマンドを実行．今回選択したテンプレートは `webpack-spa` とします．

```bash
$ npm init riot
```

ボイラープレートができたので，package.json（一部抜粋）を確認すると，

```json
"devDependencies": {
    "@riotjs/compiler": "^5.0.0", ←コンパイラモジュール
    "@riotjs/register": "^5.0.0",
    "@riotjs/webpack-loader": "^5.0.0", ←webpack プラグイン
    ...
  },
  "dependencies": {
    "@riotjs/hot-reload": "^5.0.0", ←ホットリロードモジュール
    "@riotjs/lazy": "^1.1.0",
    "@riotjs/route": "^7.0.0", ←ルーティングモジュール
    "riot": "^5.0.0" ←本体
  }
```

riot 本体とその他モジュールのバージョンが v6 に合わせてテンプレートがまだ反映されていないようですのでアップグレードします．

```bash
$ npm i riot@latest @riotjs/route@latest @riotjs/hot-reload@latest @riotjs/compiler@latest @riotjs/webpack-loader@latest
```

インストール完了後，念の為 package.json を見て更新を確認．

```json
"devDependencies": {
    "@riotjs/compiler": "^6.0.0",
    "@riotjs/register": "^5.0.0",
    "@riotjs/webpack-loader": "^6.0.0",
    ...
  },
  "dependencies": {
    "@riotjs/hot-reload": "^6.0.0",
    "@riotjs/lazy": "^1.1.0",
    "@riotjs/route": "^8.0.0",
    "riot": "^6.0.0"
  }
```

では実行します．

```bash
$ npm start
```

![](https://storage.googleapis.com/zenn-user-upload/c1fc9548edfd1ced05482e4d.png)

よし．では `TypeScript` を導入していきます．今回は `src/components/global/sidebar/sidebar.riot` ファイルを先程の例とほぼ同様に変更します．

```diff
- <script>
+ <script lang="ts">

...

  // このコンポーネントでは props がないため空オブジェクト
+ export type MyComponentProps = {}
+
+ export type MyComponentState = {
+   name: string,
+   showUser: boolean
+ }
+
+ export type MyComponent = RiotComponent<MyComponentProps, MyComponentState>
+
- export default {
+ export default withTypes<MyComponent>({

...

- }
+ })
```

では保存，実行！

…

エラー発生

```
Uncaught Error: Module parse failed: Unexpected token (4:7)
File was processed with these loaders:
 * ./node_modules/@riotjs/webpack-loader/dist/riot-webpack-loader.cjs.js
You may need an additional loader to handle the result of these loaders.
| import User from '../../includes/user/user.riot'
|
> export type MyComponentProps = {}
|
| export type MyComponentState = {
```

こちらは現在原因調査中です．ぱっと見た感じでは `@riotjs/webpack-loader` の原因っぽいですが，海外の方のエラーや公式の [Discord](https://discord.com/invite/PagXe5Y) チャンネルでのやり取りを見たところ．`babel` か `preset` 周りの設定か対応不足ではないかな？とも思えています．

> SyntaxError: Unexpected token 'export'

というエラーも出てますので．ここは後ほど更新したいと思います．

# issues

やはりリリースしたばかりのため，GitHub でも [issue](https://github.com/riot/cli/issues/51) が上がってました．プロダクションで使うにはまだ時間がかかりそうですね 💦

ではでは．
