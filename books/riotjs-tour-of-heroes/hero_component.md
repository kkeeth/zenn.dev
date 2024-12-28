---
title: 'Chapter2 ヒーローコンポーネント'
---

今回はより具体的にヒーロー（`heroes`）コンポーネントを作成し，ヒーローの一覧を表示します．

それではやっていきましょう！

# 全体構成を Angular チュートリアルに寄せる

では早速 `heroes` コンポーネントを作っていきたいのですが，今の riot のアプリケーションの全体構成が，Angular のチュートリアルと差分があるため，修正します．

## ベースコンポーネントの作成

まずは，アプリケーションのコアとなるベースコンポーネントを作成します．`app.riot` というファイルを作成し，以下を追記してください．

```html
<app>
  <div class="container">
    <h1>{ props.title }</h1>
  </div>
</app>
```

続いて，`index.html` を以下のように修正します．

```diff
  <body>
-   <div class="container">
+   <div id="root"></div>
+   <aside is="sidebar" data-riot-component class="column"></aside>
-     <div class="row">
-       <main class="column column-60">
-         <h1>Hello that's my first Riot.js App</h1>
-
-         <div
-           is="my-component"
-           data-riot-component
-           message="Hello There"
-         ></div>
-         <aside is="sidebar" data-riot-component class="column"></aside>
-       </main>
-     </div>
-   </div>
  </body>
```

さらに，`index.js` ファイルを以下のように変更します．

```diff
  import "@/style.css";
  import "@riotjs/hot-reload";
- import { mount } from "riot";
+ import { component } from "riot";
+ import App from "@/app.riot";
  import registerGlobalComponents from "@/register-global-components.js";

  // register
  registerGlobalComponents();

- // mount all the global components found in this page
- mount("[data-riot-component]");
+ // mount the root tag
+ component(App)(document.getElementById("root"), {
+   title: "Tour of Heroes with Riot"
+ });
```

## 不要なコンポーネントの削除

次に，ボイラープレートで追加された不要なコンポーネントがあるため，削除します．今回は

- `src/components/global/my-component/` ディレクトリ
- `src/components/global/sidebar/` ディレクトリ
- `src/components/includes/user/` ディレクトリ

以上 3 つが対象ですが，`global` という階層自体が不要になりますので，このディレクトリごと丸っと削除しちゃいましょう 💁

さらに，削除したファイルを読み込んでいる箇所がいくつかありますので，`index.html` ファイルを開き，以下の記述を削除してください．

```diff
  <div class="container">
    <div id="root"></div>
-   <aside is="sidebar" data-riot-component class="column"></aside>
  </div>
```

以上でお掃除も完了し，準備は完了です！

# heroes コンポーネントを作成する

それでは改めて `heroes` コンポーネントを作っていきます．

## `heroes` コンポーネントの作成

以下３つのファイルとフォルダを作成してください．

- `src/components/heroes/` ディレクトリ
- `src/components/heroes/heroes.riot` ファイル
- `src/components/heroes/heroes.spec.js` ファイル

`heroes.spec.js` についてはこの章では触れず，後ほど書いていきます（たぶん）．

では，`heroes` コンポーネントの実装に入ります．まずはヒーローの名前を表示するところまで実装してみましょう．`heroes.riot` ファイルに以下を記述してください．

```html
<heroes>
  <h2>{ hero }</h2>

  <script>
    export default {
      hero: 'Windstorm',
    };
  </script>
</heroes>
```

書けましたら，`heroes` コンポーネントを `app.riot` にてインポートし，レンダリングさせましょう．

```diff
    <h1>{ props.title }</h1>
+   <heroes />
  </div>

+ <script>
+   import Heroes from "@components//heroes/heroes.riot";
+ </script>
```

ここまでできますと，画面に `Windstorm` という名前が表示されているかと思います．

## ヒーローオブジェクトを表示する

今回のヒーローのデータには，名前の他にそれぞれのヒーローに一意な `ID` も持たせます．したがって，`heroes.riot` を以下のように変更してみましょう．

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

では最後に，ユーザーが `<input>` テキストボックスでヒーローの名前を編集できるように実装します．

まずは `heroes` コンポーネントの名前部分に入力フォームを追加しましょう．

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

名前の部分にテキストボックスが実装されましたが，このままですと名前を変更しても，タイトル下の名前に反映されませんので，これをリアルタイムで反映されるように修正します．

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

以上で Chapter2「ヒーローエディタ」は完了です！何かわからないことがあれば，遠慮なくコメントしてください！
