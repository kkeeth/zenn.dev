---
title: "Part2 ヒーローコンポーネント"
---

[前回](/books/riotjs-tour-of-heroes/environment.md) に引き続き，今回はより具体的にヒーローコンポーネントを作成していきます．

では今回もやっていきましょう！

# heroes コンポーネントを作成する

では早速コンポーネントを作っていきたいところですが，ボイラープレートのデフォルトのコンポーネントでは名前が適切でないので変更していきます．

## ディレクトリ・ファイルのリネーム

まずはいくつかのディレクトリやファイルのリネームを行います．`src/components/global` ディレクトリ以下の `my-component` と名前がつくものを全て `heroes` に変更します．具体的には以下となります．

- `src/components/components/my-component` ディレクトリ
- `src/components/components/my-component/my-component.riot` ファイル
- `src/components/components/my-component/my-component. spec.js` ファイル

さらに，`heroes.riot`, `heroes.spec.js` ファイルの `my-component` を `heroes` に変更します．

**heroes.riot**

```diff
- <my-component>
+ <heroes>
    <p>{ props.message }</p>
- </my-component>
+ </heroes>
```

**heroes.spec.js**

```diff
- import MyComponent from './my-component.riot'
+ import Heroes from './heroes.riot'
  import { expect } from 'chai'
  import { component } from 'riot'

- describe('My Component Unit Test', () => {
-   const mountMyComponent = component(MyComponent)
+ describe('Heroes Component Unit Test', () => {
+   const mountHeroes = component(Heroes)
    it('The component properties are properly rendered', () => {
      const div = document.createElement('div')

-     const component = mountMyComponent(div, {
+     const component = mountHeroes(div, {
        message: 'hello'
      })

      expect(component.$('p').innerHTML).to.be.equal('hello')
    })
  })
```

ここまで変更すると，エラーとなりますので，`index.html` を以下のように変更します．
※ `Prettier` による整形も行われています 🙇

```diff
         <main class="column column-60">
           <h1>Hello that's my first Riot.js App</h1>
-
-          <div
-            is="my-component"
-            data-riot-component
-            message="Hello There"
-          ></div>
+          <div is="heroes" data-riot-component message="Hello There"></div>
         </main>
```

## 不要なコンポーネントの削除

次に，ボイラープレートで追加されている不要なコンポーネントがあるため，削除していきます．今回は

*  `src/components/global/sidebar/` ディレクトリ
*  `src/components/includes/` ディレクトリ

ですね．こちらを丸っと削除しちゃいましょう💁

さらにこのままだとエラーになりますので，直していきます．`index.html` ファイルを開き，以下の様に修正してください．

```diff
           <h1>Hello that's my first Riot.js App</h1>
           <div is="heroes" data-riot-component message="Hello There"></div>
         </main>
-        <aside is="sidebar" data-riot-component class="column"></aside>
       </div>
     </div>
```

## `heroes` コンポーネントの実装

では，`heroes` コンポーネントを作っていきます．まずはヒーローの名前を表示するところまで実装してみましょう．`src/pages/heroes.riot` ファイルを以下のように変更してください．

```diff
  <heroes>
-   <p>{ props.message }</p>
+   <h2>{ hero }</h2>
+
+   <script>
+     export default {
+       hero: 'Windstorm'
+     }
+   </script>
  </heroes>
```

ここまでできますと，画面に `Windstorm` という名前が表示されているかと思います．

# ヒーローオブジェクトを表示する

ベースができましたので続けていきますが，今回のヒーローのデータには，名前の他にそれぞれのヒーローに一意な ID も持たせます．したがって，`heroes.riot` を以下のように変更してみましょう．

```diff
 <heroes>
-  <h2>{ hero }</h2>
+  <h2>{ hero.name } Details</h2>
+  <div><span>id: </span>{ hero.id }</div>
+  <div><span>name: </span>{ hero.name }</div>

   <script>
     export default {
-      hero: 'Windstorm'
+      hero: {
+        id: 1,
+        name: 'Windstorm'
+      }
     }
   </script>
 </heroes>
```

ここまでいきますと，以下のように ID と名前が表示されます．

![](https://storage.googleapis.com/zenn-user-upload/mpnpghzm1e1pbmimtua7zyklcyb4)

## Uppercase で表示する

今回のチュートリアルでは h2 タグの名前はアッパーケースで表示するそうなので，`heroes.riot` を以下のように変更します．

```diff
 <heroes>
-  <h2>{ hero.name } Details</h2>
+  <h2>{ hero.name.toUpperCase() } Details</h2>
   <div><span>id: </span>{ hero.id }</div>
   <div><span>name: </span>{ hero.name }</div>
```

# ヒーローを編集する

では最後に，ユーザーが `<input>` テキストボックスでヒーローの名前を編集できるように実装していきます．

まずは `heroes` コンポーネントの名前部分に入力フォームを追加していきましょう．

```diff
 <heroes>
   <h2>{ hero.name.toUpperCase() } Details</h2>
   <div><span>id: </span>{ hero.id }</div>
-  <div><span>name: </span>{ hero.name }</div>
+  <div>
+    <label for="hero-name">name:
+      <input id="hero-name" type="text" value={ hero.name } placeholder="name" />
+    </label>
+  </div>
```

これで `name:` の横の名前がテキストボックスに変更されました．

## 変更のリアルタイム反映

名前の部分にテキストボックスが実装されましたが，このままですと名前を変更しても，タイトル下の名前に反映されませんので，これをリアルタイムで反映されるように修正していきます．

```diff
   <div><span>id: </span>{ hero.id }</div>
   <div>
     <label for="hero-name">name:
-      <input id="hero-name" type="text" value={ hero.name } placeholder="name" />
+      <input
+        id="hero-name"
+        type="text"
+        value={ hero.name }
+        placeholder="name"
+        oninput={ handleInput }
+      />
     </label>
   </div>

   <script>
     export default {
       hero: {
         id: 1,
         name: 'Windstorm'
+      },
+      handleInput(e) {
+        this.hero.name = e.target.value;
+        this.update();
       }
     }
   </script>
```

ここまでできましたら，`name` のテキストボックスの値を変えてみてください．リアルタイムにタイトル下の名前も一緒に変更されると思います！

![](https://storage.googleapis.com/zenn-user-upload/q8n02fzj78mdt3v1q6o3tk5oya2e)

## タイトルとスタイルを調整

最後の最後に，今までデフォルトのままだったタイトルを（今更）今回のチュートリアルのものに変更しつつ，`ress` で初期化したスタイリングを調整していきます．

まずはタイトルを変更しますので，`index.html` を以下のように変更してください．

```diff
       <div class="row">
         <main class="column column-60">
-          <h1>Hello that's my first Riot.js App</h1>
+          <h1>Tour of Heroes with Riot</h1>
           <div is="heroes" data-riot-component message="Hello There"></div>
         </main>
       </div>
```

次にスタイリングを変更しますので，`style.css` を以下のように変更してください．

```diff
   color: #369;
   font-family: Arial, Helvetica, sans-serif;
   font-size: 250%;
+  margin: 1.6rem auto;
 }
 h2,
 h3 {
   color: #444;
   font-family: Arial, Helvetica, sans-serif;
   font-weight: lighter;
+  margin: 1.2rem auto;
 }
 body {
   margin: 2em;

...（中略）
    color: #333;
    font-family: Cambria, Georgia;
  }
+ input[type="text"] {
+   border: 1px solid gray;
+   border-radius: 2px;
+   padding: 1px 4px;
+ }

  /* everywhere else */
  * {
    font-family: Arial, Helvetica, sans-serif;
```

ここまで変更できましたら，以下の画像のように変更されていると思います．行ったことは

- タイトルと名前の上下の margin をセット
- input タグの border を調整

![](https://storage.googleapis.com/zenn-user-upload/xkzwb2vj5b59fobs2a6si9cbpzva)

以上で Part2「ヒーローエディタ」は完了です！