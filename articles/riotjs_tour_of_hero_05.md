---
title: "Riot.js で Tour of Heroes を試す - Chapter5 非同期とメッセージの作成"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["riotjs", "tutorial", "ToH"]
published: false
---

[Riot.js](https://riot.js.org/)（以下，riot）は非常にシンプルかつ軽量で入門の敷居も低く，とても書きやすいコンポーネント指向の UI ライブラリです．

[前回](https://zenn.dev/kkeeth/articles/riotjs_tour_of_hero_04) に引き続き，今回は Angular フレームワークに備わっている Service という機能に相当するコンポーネントと，メッセージを表示するコンポーネントを作成していきます．

では今回もやっていきましょう！

# サービスの追加

現在の実装では，heroes コンポーネント内で直接ヒーローデータを取得し保持していますが，コンポーネント内で直接データの取得や保存を行うべきではありません．コンポーネントはデータの受け渡しや表示に専念し，その他の処理は別ファイルへと切り出す方が望ましいです．

Angular チュートリアルの方針に沿って，アプリケーション全体でヒーローデータを取得できる `Hero` サービスを作成していきたいと思います．

## Hero サービスの作成

ということで，まずは以下のディレクトリとファイルを作成してきます．

- `src/services` ディレクトリ
- `src/services/hero.service.ts` ファイル

ファイルができましたら中身を書いていきましょう．

## ヒーローデータの取得

では，現在 `heroes` コンポーネントで取得しているヒーローデータを `hero.service` で取得するように変更していきます．

まずは `heroes.riot` ファイルから `HEROES` 配列の取得部分を削除します．

```diff
  <script>
-   import { HEROES } from '../../../mock-heroes';
    import HeroDetail from '../hero-detail/hero-detail.riot';

    export default {
      components: {
        HeroDetail
      },
-     heroes: HEROES,
      selectedHero: null,
```

そして `hero.service.ts` ファイルにて `HEROES` 配列を読み込み，コール元にレスポンスするメソッド `getHeroes()` を定義します．

```js
import { HEROES } from "../mock-heroes";

export const getHeroes = () => {
  return HEROES;
};
```

次に，`heroes.riot` にて先程定義したメソッドをマウント前に処理する `onBeforeMount()` メソッドでコールし，`heroes` 変数の初期値としてセットします．

```diff

  <script>
+   import { getHeroes } from '@/services/hero.service';
    import HeroDetail from '../hero-detail/hero-detail.riot';
    export default {
      components: {
        HeroDetail
      },
      // 厳密にはこの記述はなくても問題ない
+     heroes: [],
      selectedHero: null,
+     onBeforeMount() {
+       this.heroes = getHeroes();
+     },
      handleInput(e) {
```

以上で，ヒーローデータ取得のロジックを `heroes` コンポーネントから切り出せました．

## 非同期にデータを取得，処理

さてこれでデータ取得のロジックを分けられた様に見えますが，処理そのものを見ますと **同期的な処理になっております**．しかし，実際には外部の API を非同期的にコールしデータを取得することがほとんどでしょう（コールするメソッドのレスポンス形式が単なる JSON 形式ではなく `Promise` のことが多いので）．

したがって，この処理を非同期処理に変更します．JavaScript の `fetch API` を用いる方法もありますが，今回は有名な外部ライブラリの [axios](https://www.npmjs.com/package/axios) を用いて実装することにします．

では `axios` をインストールしましょう．ターミナルから以下のコマンドを実行してください．

```bash
$ yarn add axios
```

インストールできましたら実装していきますが，より実践的に外部からヒーローデータを JSON 形式で取得できるよう，外部サービスを用いて用意しました．

https://my-json-server.typicode.com/kkeeth/riot-tour-of-heroes/heroes

ではこちらを利用しつつ，`hero.service.ts` ファイルを以下のように変更してください．

```diff
-import { HEROES } from "../mock-heroes";
+import axios from "axios";

-export const getHeroes = () => {
-  return HEROES;
+export const getHeroes = () => {
+  return axios
+    .get(
+      "https://my-json-server.typicode.com/kkeeth/riot-tour-of-heroes/heroes"
+    )
+    .then((res) => {
+      return res.data;
+    })
+    .catch((e) => {
+      console.error(e);
+    });
 };
```

ここで注意が必要なのは，`getHeroes()` メソッドのレスポンス JSON ではなく `Promise` であるという点です．したがってコール元では `then-catch` で処理する必要があります．

ではコール元である `heroes.riot` を修正していきます．

ちなみに，一旦エラーハンドリングについてはブラウザのコンソールに出力するという簡易的な処理にとどめておきます ~~（サボったとか言わない）~~．

```diff
       components: {
         HeroDetail
       },
+      heroes: [],
       selectedHero: null,
       onBeforeMount() {
-        this.heroes = getHeroes();
+        getHeroes().then((res) => {
+          this.heroes = res
+          this.update()
+        })
       },
       handleInput(e) {
```

以上で，非同期処理へのロジック変更は完了です．実際に動作させてみると，今まで実装したものと同じ挙動をすると思います．

# メッセージの表示

では次にメッセージを表示するコンポーネントを追加してみたいと思います．やることは大きく以下の２つです．

-
