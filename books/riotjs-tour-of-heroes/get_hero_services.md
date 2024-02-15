---
title: "Part5 データ取得用のサービスの作成"
---

[Riot.js](https://riot.js.org/)（以下，riot）は非常にシンプルかつ軽量で入門の敷居も低く，とても書きやすいコンポーネント指向の UI ライブラリです．

[前回](/books/riotjs-tour-of-heroes/hero_detail.md) に引き続き，今回は Angular フレームワークに備わっている Service という機能に相当するコンポーネントと，メッセージを表示するコンポーネントを作成していきます．

では今回もやっていきましょう！

# サービスの追加

現在の実装では，heroes コンポーネント内で直接ヒーローデータを取得し保持していますが，コンポーネント内で直接データの取得や保存を行うべきではありません．コンポーネントはデータの受け渡しや表示に専念し，その他の処理は別ファイルへと切り出す方が望ましいです．

Angular チュートリアルの方針に沿って，アプリケーション全体でヒーローデータを取得できる `Hero` サービスを作成していきたいと思います．

## `Hero` サービスの作成

ということで，まずは以下のディレクトリとファイルを作成してきます．

- `src/services` ディレクトリ
- `src/services/hero.service.ts` ファイル

ファイルができましたら中身を書いていきましょう．

## ヒーローデータの取得

では，現在 `heroes` コンポーネントで取得しているヒーローデータを `hero.service` で取得するように変更していきます．

まずは `heroes.riot` ファイルから `HEROES` 配列の取得部分を削除します．

```diff
  <script>
-   import { HEROES } from "../mock-heroes";
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
import { HEROES } from "@/components/heroes/mock";

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

これで，ヒーローデータ取得のロジックを `heroes` コンポーネントから切り出せました．
以上で，非同期処理へのロジック変更は完了です．実際に動作させてみると，今まで実装したものと同じ挙動をすると思います．

では，[Part6「ダッシュボード」](/dashboard)に続きます．
