---
title: "Part3 リストの表示"
---

[前回](/books/riotjs-tour-of-heroes/hero_component.md) に引き続き，今回は複数人のヒーローをリストで表示しつつ，クリックすると ID・名前が表示される様に実装していきます．

では今回もやっていきましょう！

# 選択リストを表示する

今回の流れはざっくりと以下となります．

1. 表示するヒーローのリストもモックデータを用意
2. ヒーローのリストを表示
3. ヒーローのリストを装飾
4. ヒーローの詳細の表示/非表示を制御
5. ヒーローの詳細を装飾

## ヒーローのモックを作成する

まずはチュートリアルで用意されているデータを利用させていただきます．今回は構成はあまり考えず `src/components/global/heroes` ディレクトリ下に `mock-heroes.js` ファイルを生成し，以下を追記します．

```js
export const HEROES = [
  { id: 11, name: "Dr Nice" },
  { id: 12, name: "Narco" },
  { id: 13, name: "Bombasto" },
  { id: 14, name: "Celeritas" },
  { id: 15, name: "Magneta" },
  { id: 16, name: "RubberMan" },
  { id: 17, name: "Dynama" },
  { id: 18, name: "Dr IQ" },
  { id: 19, name: "Magma" },
  { id: 20, name: "Tornado" },
];
```

以上でモックデータの作成は完了です．

## ヒーローを表示する

では上記で作成したモックデータを読み込みます．`src/components/global/heroes.riot` ファイルに以下を追記してください．

```diff
   <script>
+    import { HEROES } from './mock-heroes';
+    console.log(HEROES);
+
     export default {
       hero: {
         id: 1,
         name: 'Windstorm'
       },
+      heroes: HEROES,
       handleInput(e) {
```

問題なければブラウザのコンソールに，モックデータが表示されると思います．

![](https://storage.googleapis.com/zenn-user-upload/s9yz51ykh0ql8rukmw6cq1wsp7cd)

モックデータが読み込めていることを確認できましたので，一覧表示していきましょう！同ファイルを以下のように変更してください．

```diff
 <heroes>
+  <h2>My Heroes</h2>
+  <ul class="heroes">
+    <li each={ hero in heroes }>
+      <button type="button">
+        <span class="badge">{ hero.id }</span>
+        <span class="name">{ hero.name }</span>
+      </button>
+    </li>
+  </ul>
+
   <h2>{ hero.name.toUpperCase() } Details</h2>
   <div><span>id: </span>{ hero.id }</div>

...(中略)

   <script>
     import { HEROES } from '../../../mock-heroes';
-    console.log(HEROES);
```

ここまで変更しますと，以下の画像のようにモックデータのヒーロー 10 人が表示されます．

![](https://storage.googleapis.com/zenn-user-upload/rzsxe9afjeehi76lgradjfsdbzh4)

ただし，この状態ですとデフォルトの見た目になってしまっているため，スタイルを調整します．同ファイルに以下を追記してください．少々長いですがご容赦ください 🙇

```diff
   </script>
+
+  <style>
+    /* HeroesComponent's private CSS styles */
+    .heroes {
+      margin: 0 0 2em 0;
+      list-style-type: none;
+      padding: 0;
+      width: 15em;
+    }
+
+    .heroes li {
+      display: flex;
+    }
+
+    .heroes button {
+      flex: 1;
+      cursor: pointer;
+      position: relative;
+      left: 0;
+      background-color: #EEE;
+      margin: .5em;
+      padding: 0;
+      border-radius: 4px;
+      display: flex;
+      align-items: stretch;
+      height: 1.8em;
+    }
+
+    .heroes button:hover {
+      color: #2c3a41;
+      background-color: #e6e6e6;
+      left: .1em;
+    }
+
+    .heroes button:active {
+      background-color: #525252;
+      color: #fafafa;
+    }
+
+    .heroes button.selected {
+      background-color: black;
+      color: white;
+    }
+
+    .heroes button.selected:hover {
+      background-color: #505050;
+      color: white;
+    }
+
+    .heroes button.selected:active {
+      background-color: black;
+      color: white;
+    }
+
+    .heroes .badge {
+      display: inline-block;
+      font-size: small;
+      color: white;
+      padding: 0.8em 0.7em 0 0.7em;
+      background-color: #405061;
+      line-height: 1em;
+      margin-right: .8em;
+      border-radius: 4px 0 0 4px;
+    }
+
+    .heroes .name {
+      align-self: center;
+    }
+  </style>
 </heroes>
```

上記が追記できますと，以下のような見た目になるかと思います．スッキリして見やすくなりましたね．

![](https://storage.googleapis.com/zenn-user-upload/9q613fep2aql0outmel58ikwsijw)

# ヒーローの詳細

さて，ヒーローのリストが表示できましたが現状ですと，リストの中のヒーローをクリックしても何も変化しません．これを，選択されたヒーローの詳細を画面下に表示する必要があります．

## クリックイベントのハンドラを追加する

まずは，各ヒーローをクリックしたときのイベントハンドラを追加します．以下を追記してください．

```diff
   <h2>My Heroes</h2>
   <ul class="heroes">
     <li each={ hero in heroes }>
-      <button type="button">
+      <button
+        type="button"
+        onclick={ handleSelect }
+        data={ hero }
+      >
         <span class="badge">{ hero.id }</span>
         <span class="name">{ hero.name }</span>
       </button>
     </li>
   </ul>

...（中断）

       handleInput(e) {
         this.hero.name = e.target.value;
         this.update();
-      }
+      },
+      handleSelect(e) {
+        this.hero.id = e.target.closest('button').data.id;
+        this.hero.name = e.target.closest('button').data.name;
+        this.update();
+      }
     }
   </script>
```

`Angular` では以下のように，クリックイベントのバインディングに指定する関数の引数に任意の値を指定できます．

```html
<button type="button" (click)="onSelect(hero)">
```

これを riot でも同様なコードを書いたとします．

```html
<button
  type="button"
  onclick={ handleSelect(hero) }
>
```

**これを実行するとエラーになってしまいます．** Riot はイベントのバインディングに指定する関数を `hoge(fuga)` のように書いてしまうと，**初期レンダリングの際に関数を評価・実行してしまう** からです．したがって，ちょっとテクニカルにはなってしまいますが，

```js
e.target.closest('button').data
```

のように，クリックイベント時のイベントオブジェクトのターゲットに最も近い `<button>` タグにアクセスし，その属性に指定されている `data` にアクセスすることで選択した `hero` を取得しています．

追記できましたら任意のヒーローをクリックしてみてください．選択したヒーローの詳細データが表示されると思います．

## 詳細セクションを更新する

次は選択したヒーローの詳細をリファクタリングしていきましょう．今まで使用してきた `hero` 変数を `selectedHero` に変更します．画面描画時は何も選択されていないので初期値は `{}` とします．

```diff
   </ul>

-  <h2>{ hero.name.toUpperCase() } Details</h2>
-  <div><span>id: </span>{ hero.id }</div>
+  <h2>{ selectedHero.name.toUpperCase() } Details</h2>
+  <div><span>id: </span>{ selectedHero.id }</div>
   <div>
     <label for="hero-name">name:
       <input
         id="hero-name"
         type="text"
-        value={ hero.name }
+        value={ selectedHero.name }
         placeholder="name"
         oninput={ handleInput }
       />

...（中断）

     import { HEROES } from './mock-heroes';

     export default {
-      hero: {
-        id: 1,
-        name: 'Windstorm'
-      },
       heroes: HEROES,
+      selectedHero: {},
       handleInput(e) {
-        this.hero.name = e.target.value;
+        this.selectedHero.name = e.target.value;
         this.update();
       },
       handleSelect(e) {
-        this.hero.id = e.target.closest('button').data.id;
-        this.hero.name = e.target.closest('button').data.name;
+        this.selectedHero.id = e.target.closest('button').data.id;
+        this.selectedHero.name = e.target.closest('button').data.name;
         this.update();
       }
```

ここまで変更しますと，ブラウザのコンソールにエラーが表示されるはずです．理由はもちろん `selectedHero` が空オブジェクトのため， `name` プロパティにアクセスしようとしても `undefined` となり，`toUpperCase()` メソッドで変更しようとしてもできないからです．

したがって，初期値がセットされていない場合はそもそも詳細部分をレンダリングしないように変更します．

```diff
     </li>
   </ul>

-  <h2>{ selectedHero.name.toUpperCase() } Details</h2>
-  <div><span>id: </span>{ selectedHero.id }</div>
-  <div>
-    <label>name:
-      <input
-        type="text"
-        value={ selectedHero.name }
-        placeholder="name"
-        oninput={ handleInput }
-      />
-    </label>
+  <div if={ selectedHero.id }>
+    <h2>{ selectedHero.name.toUpperCase() } Details</h2>
+    <div><span>id: </span>{ selectedHero.id }</div>
+    <div>
+      <label>name:
+        <input
+          type="text"
+          value={ selectedHero.name }
+          placeholder="name"
+          oninput={ handleInput }
+        />
+      </label>
+    </div>
   </div>

   <script>
```

ここまで変更できましたら再度画面を更新し，適当にヒーローを選択してみてください．選択したヒーローごとに詳細のデータが変更されると思います．また `name` のテキストボックスを変更していただくと，リストの表示も詳細のタイトルもリアルタイムで値が反映されると思います．

## 選択したヒーローの装飾

本記事の最後に，選択したヒーローがリストのどれなのか分かりづらいので，少々装飾します．この実装は簡単で，`hero` 変数と `selectedHero` 変数の中身が同じであれば，`selected` という class を対象の `<li>` に付与してあげれば解決です．
※ちなみに，`selected` の CSS スタイリングはすでに実装済みです．

```diff
     <button
       type="button"
+      class={ hero.id === selectedHero.id && 'selected' }
       onclick={ handleSelect }
       data={ hero }
     >
```

ここまで変更できましたら，以下の画像のように選択したヒーローの見た目が変わるはずです．

![チャプター3の完成イメージ](https://storage.googleapis.com/zenn-user-upload/534855f68ae3-20240709.png)

以上で Part3「リストの表示」は完了です．何かわからないことがあれば，遠慮なくコメントしてください！