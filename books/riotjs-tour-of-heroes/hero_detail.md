---
title: "Part4 フィーチャーコンポーネントの作成"
---

[前回](/books/riotjs-tour-of-heroes/show_list.md) に引き続き，今回はリファクタリングの回となります．機能としては前回のものから何も真新しいものはなく，動作も全く同じですが，コンポーネント単位でファイルを分割することで保守性・拡張性の向上が望めます．

では今回もやっていきましょう！

# hero-detail コンポーネントを作成する

それではまずはヒーローの詳細を表示するためのコンポーネントと，それを格納するディレクトリを作成していきます．具体的には以下です．
※`spec.js` ファイルはテスト用ファイルですが，後の回にて説明しますので，一旦空ファイルのままで良いです．

- `src/components/heroes/hero-detail` ディレクトリ
- `src/components/heroes/hero-detail/hero-detail.riot` ファイル

作成できましたら，それぞれのファイルの中身を書いていきましょう．

## template を記述する

では `hero-detail.riot` ファイルに追記していきます．こちらは既存の `heroes.riot` ファイルのヒーローの詳細を表示する部分を丸っと持っていきます．

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
-          placeholder="name"
-          value={ selectedHero.name }
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
  <div if={ selectedHero.id }>
    <h2>{ selectedHero.name.toUpperCase() } Details</h2>
    <div>
      <span>id: </span>{ selectedHero.id }
    </div>
    <div>
      <label for="hero-name">name:
        <input
          id="hero-name"
          type="text"
          value={ selectedHero.name }
          placeholder="name"
          oninput={ handleInput }
        />
      </label>
    </div>
  </div>

  <script>
    export default {
      selectedHero: {},
      onBeforeUpdate(props) {
        this.selectedHero = props.hero;
      },
    };
  </script>
</hero-detail>
```

`hero-detail` コンポーネントを記述するに当たり，`props` で表示する `hero` オブジェクトを取得できるように設定もしております．初期レンダリング時はヒーローは選択されていないので初期は null でよく，また`onMounted()` の処理も不要となります．

# hero-detail コンポーネントを表示する

それでは作成した `hero-detail` コンポーネントを表示していきましょう．`heroes` コンポーネントにて，`hero-detail` コンポーネントを読み込み，配置します．

```diff
     </li>
   </ul>
+  <hero-detail hero={ selectedHero } />

   <script>
     import { HEROES } from '../../../mock-heroes';
+    import HeroDetail from '../hero-detail/hero-detail.riot';
```

読み込みと一緒に，`hero` というキーで `props` として選択した `selectedHero` オブジェクトを渡しています．ここまでできますと，画面からも選択したヒーローの詳細情報が表示されるようになります．

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

入力したヒーローの名前を `hero-detail` コンポーネントにて更新するだけなら `handleInput` の変更は不要ですが，ヒーローリストの名前の部分にも適用させたいので，上記のように `this.heroes` を走査して合致するヒーローの名前を更新しました💁

ここまでできましたら入力フォームの値を自由に変更すると，タイトルの部分と上部のリストの名前も同時に変更されるかと思いますのでご確認ください．

![詳細と一覧のヒーローの名前を変更](https://storage.googleapis.com/zenn-user-upload/773a9e71989b-20240709.png)

以上で Part4「フィーチャーコンポーネントの作成」は完了です．何かわからないことがあれば，遠慮なくコメントしてください！可能な限りご説明させていただきます！
