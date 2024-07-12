---
title: "Chapter2 ヒーローコンポーネント"
---

今回はより具体的にヒーローコンポーネントを作成していきます．

# heroes コンポーネントを作成する

では早速 `heroes` コンポーネントを作っていきますが，厳密には `heroes` ページ（コンポーネント）を作る形になります．

## ディレクトリ・ファイルのリネームと修正

まずはいくつかのディレクトリやファイルのリネームを行います．`src/pages/home.riot` ファイルの名前を `heroes.riot` に変更し，さらに，`app.riot`，`heroes/riot`，`src/pages.js` ファイルを以下のように変更します．

**app.riot**
```diff
   Router,
   Route,
   NotFound,
-  Home: lazy(Loader, () => import(
-    /* webpackPrefetch: true, webpackChunkName: 'pages/home' */
-    './pages/home.riot'
+  Heroes: lazy(Loader, () => import(
+    /* webpackPrefetch: true, webpackChunkName: 'pages/heroes' */
+    './pages/heroes.riot'
   )),
   About: lazy(
```

**heroes.riot**
```diff
-<home>
+<heroes>
   <h1>I am the home page</h1>
   <p>Welcome</p>
-</home>
+</heroes>
```

**pages.js**
```diff
 export default [
   {
     path: "/",
-    label: "Home",
-    componentName: "home",
+    label: "Heroes",
+    componentName: "heroes",
   },
```

一旦 `heroes` コンポーネントの作成は完了です．

## 不要なコンポーネントの削除

次に，ボイラープレートで追加されている不要なコンポーネントがあるため，削除していきます．今回は

* `src/components/global/my-component/` ディレクトリ
*  `src/components/global/sidebar/` ディレクトリ
*  `src/components/includes/user/` ディレクトリ

ですね．こちらを丸っと削除しちゃいましょう💁

さらに，削除したファイルを読み込んでいる箇所がいくつかありますので，`app.riot` ファイルを開き，以下の記述を削除してください．

```html
<!-- notice how <sidebar> is registered as global component -->
<div class="column column-40">
 <sidebar/>
</div>
```

以上でお掃除完了です．

## `heroes` コンポーネントの実装

では，`heroes` コンポーネントの実装に入ります．まずはヒーローの名前を表示するところまで実装してみましょう．`heroes.riot` ファイルを以下のように変更してください．

```diff
  <heroes>
-   <h1>I am the home page</h1>
-   <p>Welcome</p>
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

ベースができましたので続けていきますが，今回のヒーローのデータには，名前の他にそれぞれのヒーローに一意な `ID` も持たせます．したがって，`heroes.riot` を以下のように変更してみましょう．

```diff
 <heroes>
-  <h2>{ hero }</h2>
+  <h2>{ hero.name.toUpperCase() } Details</h2>
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

ここまでいきますと，以下のように ID と名前が表示されます．今回のチュートリアルでは h2 タグの名前はアッパーケースで表示するそうなので，`toUpperCase()` を利用しています．


![ヒーローオブジェクトの表示](/images/books/riotjs_toh/02_show_hero_object.png)


# ヒーローを編集する

では最後に，ユーザーが `<input>` テキストボックスでヒーローの名前を編集できるように実装していきます．

まずは `heroes` コンポーネントの名前部分に入力フォームを追加していきましょう．

```diff
 <heroes>
   <h2>{ hero.name.toUpperCase() } Details</h2>
   <div><span>id: </span>{ hero.id }</div>
-  <div><span>name: </span>{ hero.name }</div>
+  <div>
+    <label for="hero-name">name:</label>
+    <input id="hero-name" type="text" value={ hero.name } placeholder="name" />
+  </div>
```

これで `name:` の横の名前がテキストボックスに変更されました．

## 変更のリアルタイム反映

名前の部分にテキストボックスが実装されましたが，このままですと名前を変更しても，タイトル下の名前に反映されませんので，これをリアルタイムで反映されるように修正していきます．

```diff
   <div><span>id: </span>{ hero.id }</div>
   <div>
     <label for="hero-name">name:</label>
-    <input id="hero-name" type="text" value={ hero.name } placeholder="name" />
+    <input
+      id="hero-name"
+      type="text"
+      value={ hero.name }
+      placeholder="name"
+      oninput={ handleInput }
+    />
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

![ヒーローネームの変更動画](/images/books/riotjs_toh/02_change_hero_name.gif)

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

以上で Chapter2「ヒーローエディタ」は完了です！