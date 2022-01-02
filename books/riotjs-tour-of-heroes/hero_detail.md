---
title: 'Part4 フィーチャーコンポーネントの作成'
---

[Riot.js](https://riot.js.org/)（以下，riot）は非常にシンプルかつ軽量で入門の敷居も低く，とても書きやすいコンポーネント指向の UI ライブラリです．

[前回](https://zenn.dev/kkeeth/articles/riotjs_tour_of_hero_03) に引き続き，今回はリファクタリングの回となります．機能としては前回のものから何も真新しいものはなく，動作も全く同じですが，コンポーネント単位でファイルを分割することで保守性・拡張性の向上が望めます．

では今回もやっていきましょう！

# hero-detail コンポーネントを作成する

それではまずはヒーローの詳細を表示するためのコンポーネントと，それを格納するディレクトリを作成していきます．具体的には以下です．
※`spec.js` ファイルはテスト用ファイルですが，後の回にて説明しますので，一旦空ファイルのままで良いです．

- `src/components/global/hero-detail` ディレクトリ
- `src/components/global/hero-detail/hero-detail.riot` ファイル
- `src/components/global/hero-detail/hero-detail.spec.js` ファイル

作成できましたら，それぞれのファイルの中身を書いていきましょう．

## template を記述する

では `hero-detail.riot` ファイルに追記していきます．こちらは既存の `heroes.riot` ファイルのヒーローの詳細を表示する部分を丸っと持っていきますが，このとき `selectedHero` という変数名を `hero` に戻しておきます．

**heroes.riot**

```diff
     </li>
   </ul>

-  <div if={ selectedHero }>
-    <h2>{ selectedHero.name.toUpperCase() } Details</h2>
-    <div><span>id: </span>{ selectedHero.id }</div>
-    <div>
-      <label>name:
-        <input
-          type="text"
-          value={ selectedHero.name }
-          placeholder="name"
-          oninput={ handleInput }
-        />
-      </label>
-    </div>
-  </div>
-
   <script>
```

**hero-detail.riot**

```html
<hero-detail>
  <div if={ hero.id }>
    <h2>{ hero.name.toUpperCase() } Details</h2>
    <div><span>id: </span>{ hero.id }</div>
    <div>
      <label>
        name:
        <input
          type="text"
          value={ hero.name }
          placeholder="name"
          oninput={ handleInput }
        />
      </label>
    </div>
  </div>

  <script>
    export default {
      hero: {},
      onBeforeUpdate(props) {
        this.hero = props.hero
      },
    }
  </script>
</hero-detail>
```

`hero-detail` コンポーネントを記述するに当たり，`props` で表示する `hero` オブジェクトを取得できるように設定もしております．初期レンダリング時はヒーローは選択されていないので初期は null でよく，また`onMounted()` の処理も不要となります．

# hero-detail コンポーネントを表示する

それでは作成した `hero-detail` コンポーネントを表示していきましょう．まずは `heroes` コンポーネントにて読み込みます．

```diff
     </li>
   </ul>
+  <hero-detail hero={ selectedHero } />

   <script>
     import { HEROES } from '../../../mock-heroes';
+    import HeroDetail from '../hero-detail/hero-detail.riot';

     export default {
+      components: {
+        HeroDetail
+      },
       heroes: HEROES,
```

読み込みと一緒に，`hero` というキーで `props` として選択したヒーロー `selectedHero` オブジェクトを渡しています．ここまでできますと，画面からも選択したヒーローの詳細情報が表示されるようになります．

![](https://storage.googleapis.com/zenn-user-upload/5wii4nxtx04t18dj4gftq848ies8)

## ヒーローの名前を変更できるように

ヒーローの詳細を表示はできましたが，現状では `name` のテキストボックスを変更しても画面には反映されません．

というのも，`hero-detail` コンポーネントの 11 行目で `oninput` イベントハンドラとして `handleInput` というハンドラをセットしていますが，こちらのハンドラは元々 `heroes` コンポーネントで生成しており，これを `hero-detail` コンポーネントに渡せていないので，こちらを修正していきましょう．

**hero-detail.riot**

```diff
           value={ hero.name }
           placeholder="name"
-          oninput={ handleInput }
+          oninput={ props.handleInput }
         />
       </label>
```

**heroes.riot**

```diff
     </li>
   </ul>
-  <hero-detail hero={ selectedHero } />
+  <hero-detail hero={ selectedHero } handle-input={ handleInput } />

   <script>
     import { HEROES } from '../../../mock-heroes';

     (中略)

     handleInput(e) {
       this.selectedHero.name = e.target.value;
+      this.heroes.forEach((item) => {
+        if (item.id === this.selectedHero.id) {
+          item.name = e.target.value;
+        }
+      });
       this.update();
     },
```

ここまでできましたら，前回と同じ状態のアプリケーションが出来上がっているかと思いますが，入力フォームの値を自由に変更すると，タイトルの部分と上部のリストの名前も同時に変更されるかと思いますのでご確認ください．

![](https://storage.googleapis.com/zenn-user-upload/pxgp3khy914o4gmnr73j0hndgwqp)

以上で Part4「フィーチャーコンポーネントの作成」は完了です．何かわからないことがあれば，遠慮なくコメントしてください！可能な限りご説明させていただきます！

では，Part5「非同期とメッセージの作成」に続きます．
