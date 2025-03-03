---
title: 'Chapter6 ナビゲーションの追加'
---

ここまでの実装で，アプリケーションとしては一通り実装はできています．しかし全体構造として，1 画面にすべてのコンポーネントがレンダリングされ，DOM としては存在している状態です．ヒーローズビューとダッシュボードビューの間で行き来できる用拡張することで，選択したヒーローの詳細ビューを表示できます．

それではやっていきましょう！

# ルーティングの機構を整える

この変更が実現すると，以下の図のようにアプリケーションを行き来できるようになります．

![View Navigation](https://angular.jp/generated/images/guide/toh/nav-diagram.png)

参照：Angular の公式チュートリアル「Tour of Heroes」の「ルーティングを使ったナビゲーションの追加」の章より

## `@riotjs/route` の導入とルーティングの設定

これを実現するためには，ルーティングの機構を導入する必要があります．riot では `@riotjs/route` というコアライブラリが有りますので，こちらをインストールします．

```bash
npm install -S @riotjs/route
```

:::message
`riot/create-riot` のバージョンや，開発環境の作り方によっては始めからインストールされている可能性があります．`package.json` の中をご確認いただくか，

```bash
npm ls --depth=0
```

コマンドから `@riotjs/route` がないかご確認ください．
:::

インストールできましたら，実際に `app.riot` に組み込んでいきましょう 🙋‍♂

```diff
  <div class="container">
    <h1>{ props.title }</h1>
-   <heroes />
+   <router>
+     <route path="/">
+       <main is="heroes" />
+     </route>
+   </router>
    <messages />
  </div>

  <script>
+   import { Router, Route, route, toRegexp, match } from '@riotjs/route'
    import Heroes from "@components/heroes/heroes.riot";
    import Messages from '@components/messages/messages.riot';

+   export default {
+     components: {
+       Router,
+       Route,
+     }
+   }
  </script>
```

詳しい使い方は [`@riotjs/route`](https://github.com/riot/route) 公式の README にお譲りするとして，インポートした `Router, Route` コンポーネントを使ってルーティングの機構を組み込んでいます．

- `Router`: このコンポーネント内のリンククリックを自動的に検知し， `riotjs/router` のルートイベントを発火させる．コンポーネント外はブラウザ標準のリンクとなる
- `Route`: 子コンポーネントで URL パラメータとクエリを検出できるようにする．`path` で指定した文字列（正規表現で指定も可）と URL のパスが一致したものがレンダリングされる

上記の実装だと，`/` の場合 `heroes` コンポーネントがレンダリングされます．これを，`/heroes` にアクセスしたときに `heroes` コンポーネントがレンダリングされること，ナビゲーションリンクも実装しましょう．

```diff
  <div class="container">
    <h1>{ props.title }</h1>
　   <router>
+     <div class="row">
+       <nav class="menu column">
+         <a href="/heroes">Heroes</a>
+       </nav>
+     </div>
-     <route path="/">
+     <route path="/heroes">
　       <main is="heroes" />
　     </route>
　　  </router>
```

ここまで行くと，初期画面は以下のようにヒーローは表示されず，`heroes` コンポーネントへのリンクが表示され，クリックすると URL が `/heroes` となり，ヒーローの一覧が表示されるようになります．

![ナビゲーション実装後の初期画面](/images/books/riotjs_toh/06_routing_heroes.png)

## ダッシュボードビュー（`dashboard`） を追加

まずは必要なフォルダとファイルを作ります．

- `src/components/dashboard/`
- `src/components/dashboard/dashboard.riot`
- `src/components/dashboard/dashboard.spec.js`

作成できましたら，今まで通り `.spec.js` ファイルは後回しにして，`.riot` ファイルの方に以下を追記してください．

```html
<dashboard>
  <h2>Top Heroes</h2>
  <div class="heroes-menu">
    <a each="{" hero in heroes }> { hero.name } </a>
  </div>

  <script>
    import heroService from '@services/hero.service';

    export default {
      heroes: [],
      onBeforeMount() {
        heroService.on('heroesUpdated', (heroes) => {
          this.heroes = heroes.slice(1, 5);
        });
        heroService.getHeroes();
      },
    };
  </script>
</dashboard>
```

今回は，初期レンダリング時のデータの取得もまとめて実装してしまっています．今回はヒーローリストの `ID（数字）` が若い方から４ヒーローを取得しています．

## `dashboard` とデフォルトのルートを設定

続いて，`dashboard` コンポーネント用のナビゲーションリンクと，レンダリングの設定をしましょう．ついでに，デフォルトのルート `/` の際も `dashboard` コンポーネントが表示されるように設定します．

```diff
       <div class="row">
         <nav class="menu column">
+          <a href="/dashboard">Dashboard</a>
           <a href="/heroes">Heroes</a>
         </nav>
       </div>
+      <route path="/(dashboard)?">
+        <main is="dashboard" />
+      </route>
       <route path="/heroes">
         <main is="heroes" />
       </route>
```

`route` の `path` 指定を正規表現で `/` と `/dashboard` のどちらも検知し `dashboard` コンポーネントを描画するように指定しています．

## ナビゲーションリンクのスタイリング

現在だと `<nav>` のスタルがちょっとイマイチのため，微調整します．`app.riot` に以下を追記してください．

```diff
     </script>
+
+   <style>
+     /* AppComponent's private CSS styles */
+     h1 {
+       margin-bottom: 0;
+     }
+     nav a {
+       padding: 1rem;
+       text-decoration: none;
+       margin-top: 10px;
+       display: inline-block;
+       background-color: #e8e8e8;
+       color: #3d3d3d;
+       border-radius: 4px;
+     }
+     nav a + a {
+       margin-left: 10px;
+     }
+
+     nav a.active {
+       color: white;
+       background-color: black;
+     }
+     nav a:hover {
+       color: white;
+       background-color: #42545C;
+     }
+   </style>
  </app>
```

すると，以下のような見た目になっているはずです．

![ナビゲーションのスタイリング後](/images/books/riotjs_toh/06_styled_navigation.png)

## `dashboard` コンポーネントのスタイリング

現状ですと，`dashboard` コンポーネントのヒーロー名の表示が見辛いので，スタイリングをしましょう．`dashboard.riot` に以下を追記してください．

```diff
      }
    </script>
+   <style>
+     /* DashboardComponent's private CSS styles */
+
+     h2 {
+       text-align: center;
+     }
+
+     .heroes-menu {
+       padding: 0;
+       margin: auto;
+       max-width: 1000px;
+
+       /* flexbox */
+       display: flex;
+       flex-direction: row;
+       flex-wrap: wrap;
+       justify-content: space-around;
+       align-content: flex-start;
+       align-items: flex-start;
+     }
+
+     a {
+       background-color: #3f525c;
+       border-radius: 2px;
+       padding: 1rem;
+       font-size: 1.2rem;
+       text-decoration: none;
+       display: inline-block;
+       color: #fff;
+       text-align: center;
+       width: 100%;
+       min-width: 70px;
+       margin: .5rem auto;
+       box-sizing: border-box;
+
+       /* flexbox */
+       order: 0;
+       flex: 0 1 auto;
+       align-self: auto;
+     }
+
+     @media (min-width: 600px) {
+       a {
+         width: 18%;
+         box-sizing: content-box;
+       }
+     }
+
+     a:hover {
+       background-color: #000;
+     }
+   </style>
  </dashboard>
```

ここまで実装できますと，以下のような見た目になっているかと思います．

![ダッシュボードコンポーネントの表示](/images/books/riotjs_toh/06_show_dashboard.png)

# ヒーローの詳細へ遷移

選択したヒーローの詳細情報を表示する `hero-detail` コンポーネントは，現状ユーザーは 3 つの方法でアクセスできるべきです．

- ダッシュボードのヒーローをクリック
- ヒーローリストのヒーローをクリック
- 表示するヒーローを識別するブラウザのアドレスバーに"ディープリンク"URL を貼り付ける

これを実現し，`hero-detail` コンポーネントを `heroes` コンポーネントから独立させます．

## `heroes` コンポーネントから `hero-detail` コンポーネントを削除

まずはエイヤで，`hero-detail` コンポーネントから `heroes` コンポーネントから引っ剥がしちゃいます．

```diff
   </ul>

-  <hero-detail hero={ selectedHero } handle-input={ handleInput } />
-
   <script>
     import heroService from '@services/hero.service';
     import messageService from '@services/message.service';
-    import HeroDetail from '@components/hero-detail/hero-detail.riot';

     export default {
       // （中略）
-      handleInput(e) {
-        this.selectedHero.name = e.target.value;
-        this.heroes.forEach((item) => {
-          if (item.id === this.selectedHero.id) {
-            item.name = e.target.value;
-          }
-        });
-        this.update();
-      },
```

ただし，`handleInput` メソッドは `hero-detail` コンポーネントで引き続き利用するため，こちらは移植してしまいます．

```diff
           value={ selectedHero.name }
           placeholder="name"
-          oninput={ props.handleInput }
+          oninput={ handleInput }
         />
       </label>
     </div>

// （中略）

         heroService.getHero(Number(props.id));
       },
+      handleInput(e) {
+        this.selectedHero.name = e.target.value;
+        this.update();
+      }
     };
```

## `hero-detail` ルートを追加する

`~/hero-detail/13` などの URL でアクセスすると，ID が 13 のヒーローの詳細が表示されるように，ルートを追加します．`app.riot` を以下のように変更してください．

```diff
    <route path="/heroes">
      <main is="heroes" />
    </route>
+   <route path="/hero-detail/:id">
+     <main is="hero-detail" id={ route.params.id } />
+   </route>
  </router>

// （中略）

  <script>
    import Heroes from "@components/heroes/heroes.riot";
    import Messages from '@components/messages/messages.riot';
+   import HeroDetail from "@components/hero-detail/hero-detail.riot";
    import { Router, Route, route, toRegexp, match } from '@riotjs/route'
```

一旦こちらで設定は完了です．

## `dashboard`, `heroes` コンポーネントのヒーローのリンク

設定はしましたが，今は URL で直接アクセスするしかヒーローの詳細が表示されないため，`dashboard`, `heroes` コンポーネントのヒーローのリンクを修正します．

**dashboard.riot**

```diff
   <div class="heroes-menu">
-    <a each={ hero in heroes }>
+    <a each={ hero in heroes } href={ `/hero-detail/${hero.id}` }>
       { hero.name }
     </a>
```

**heroes.riot**

```diff
     <li each={ hero in heroes }>
-      <button
-        type="button"
-        class={ hero.id === selectedHero.id && 'selected' }
-        onclick={ handleSelect }
-        data={ hero }
-      >
+      <a href={`/hero-detail/${hero.id}`}>
         <span class="badge">{ hero.id }</span>
         <span class="name">{ hero.name }</span>
-      </button>
+      </a>
     </li>
```

これでヒーローの詳細画面にアクセスすることができました．しかしまだ詳細情報自体は表示されていませんので，後ほど修正します．

`heroes` コンポーネントが `<button>` から `<a>` に変わったためスタイルが崩れてしまいました．`heroes.riot` の CSS を修正しましょう．差分が多いので，以下で丸っと上書きしていただくのが良さそうです！

```css
/* HeroesComponent's private CSS styles */
.heroes {
  margin: 0 0 2em 0;
  list-style-type: none;
  padding: 0;
  width: 15em;
}
.heroes li {
  position: relative;
  cursor: pointer;
}

.heroes li:hover {
  left: 0.1em;
}

.heroes a {
  color: #333;
  text-decoration: none;
  background-color: #eee;
  margin: 0.5em;
  padding: 0.3em 0;
  height: 1.6em;
  border-radius: 4px;
  display: block;
  width: 100%;
}

.heroes a:hover {
  color: #2c3a41;
  background-color: #e6e6e6;
}

.heroes a:active {
  background-color: #525252;
  color: #fafafa;
}

.heroes .badge {
  display: inline-block;
  font-size: small;
  color: white;
  padding: 0.8em 0.7em 0 0.7em;
  background-color: #405061;
  line-height: 1em;
  position: relative;
  left: -1px;
  top: -4px;
  height: 1.8em;
  min-width: 16px;
  text-align: right;
  margin-right: 0.8em;
  border-radius: 4px 0 0 4px;
}
```

## `heroes` コンポーネントの不要なコードの削除

`heroes` コンポーネントで既に使わなくなったメソッドやコードがあるので，こちらも削除してしまいましょう．

```diff
  <script>
    import heroService from '@services/hero.service';
-   import messageService from '@services/message.service';

    export default {
      heroes: [],
-     selectedHero: {},
      onBeforeMount() {
        heroService.on('heroesUpdated', (heroes) => {
          this.heroes = heroes;
        });
        heroService.getHeroes();
      },
-     handleSelect(e) {
-       this.selectedHero.id = e.target.closest('button').data.id;
-       this.selectedHero.name = e.target.closest('button').data.name;
-       messageService.add(`HeroesComponent: Selected hero id=${this.selectedHero.id}`);
-       this.update();
-     }
```

以上でお掃除完了です！

## `hero-detail` コンポーネントでヒーローデータを取得

以前は，親の `heroes` コンポーネントが `hero-detail` コンポーネントのプロパティを設定しており`hero-detail` コンポーネントは単に受け取ったデータを表示していました．

しかし，`heroes` コンポーネントはその責務から開放され，今はルーターが `~/hero-detail/11` のような URL に応答して `hero-detail` コンポーネントを作成します．

したがって，`hero-detail` コンポーネントが渡された ID をキーに `Hero` サービスからデータを取得するように変更します．

```diff
   <script>
+    import heroService from '@services/hero.service';
+
     export default {
       selectedHero: {},
-      onBeforeUpdate(props) {
-        this.selectedHero = props.hero;
+      onBeforeMount(props) {
+        heroService.on('getHero', (hero) => {
+          this.selectedHero = hero;
+        })
+
+        heroService.getHero(Number(props.id));
       },
       handleInput(e) {
         this.selectedHero.name = e.target.value;
         this.update();
       }
     };
   </script>
```

さらに，`hero.service.js` にて ID を指定してヒーローを取得するためのメソッドを実装します．

```diff
       console.error('Failed to fetch heroes:', error);
     }
+  },
+  async getHero(id) {
+    try{
+      // const response = await fetch(`https://api.+xample.com/hero/${id}`);
+      // const heroes = await response.json();
+      const hero = HEROES.find(h => h.id === id);
+      messageService.add(`HeroService: fetched hero id=${id}`);
+      this.trigger('getHero', hero)
+    } catch (error) {
+      console.error('Failed to fetch heroes:', error);
+    }
   }
```

今回は引数に `id` を受け取り，`HEROES` 配列からその ID と合致するヒーローのデータをフィルタリングし，observable のイベントトリガーで渡しています．

実際のアプリケーションでは，`id` を指定して取得する API のエンドポイントが用意されていると思いますので，上記のようなフィルタリング処理は不要かもしれませんね 💁

## 戻るボタンの追加

現在は `hero-detail` コンポーネントを表示した後，ブラウザの戻るボタンをクリックすればヒーローリストまたはダッシュボード画面に戻れます．これを `hero-detail` コンポーネントにもそうできるボタンを用意しましょう．

```diff
+        oninput={ handleInput }
+      />
     </div>
+    <button type="button" onclick={ goBack }>go back</button>
   </div>

// （中略）

     handleInput(e) {
       this.selectedHero.name = e.target.value;
       this.update();
     },
+    goBack() {
+      history.back();
+    }
   };
```

ブラウザを更新してクリックしてみましょう．ユーザーは新しいボタンを使って，アプリケーション内を移動できるようになりました．

`Angular` には [location](https://angular.jp/api/common/Location) というブラウザと対話するためのサービスがあります．これは前のビューに戻ることができるサービスです．

riot には上記のようなものは存在しないので，シンプルにブラウザの History API を利用して実装しました．

## `hero-detail` のスタイリング

最後にスタイルの微調整をします．`hero-detail` コンポーネントの `<style>` を以下で丸っと上書きしてください．

```css
/* HeroDetailComponent's private CSS styles */
label {
  color: #435960;
  font-weight: bold;
}
input {
  font-size: 1em;
  padding: 0.5rem;
}
button {
  margin-top: 20px;
  background-color: #eee;
  padding: 1rem;
  border-radius: 4px;
  font-size: 1rem;
}
button:hover {
  background-color: #cfd8dc;
}
button:disabled {
  background-color: #eee;
  color: #ccc;
  cursor: auto;
}
```

ここまでできましたら，アプリケーションを更新してみてみましょう．

**ダッシュボード**
![実装完了（ダッシュボード）](/images/books/riotjs_toh/06_completed_dashboard.png)

**ヒーロー一覧**
![実装完了（ヒーロー一覧）](/images/books/riotjs_toh/06_completed_heroes.png)

**ヒーロー詳細**
![実装完了（ヒーロー詳細）](/images/books/riotjs_toh/06_completed_hero-detail.png)

# 諸々のリファクタリング

本章で実装したかったことは完了ですが，後は全体的なリファクタリングをしたいと思います．

## ルートの一覧をまとめる

現状は，ルートごとに一つ一つ手書きで `<route>`　を書いておりますが，ルートの一覧をファイルに纏めてしまい，マークアップはループで書いてしまう方が管理が楽になります．

`routes.js` ファイルを作成し，以下を記述してください．

```js
export default [
  {
    path: '/',
    label: 'Dashboard',
    componentName: 'dashboard',
    nav: false,
  },
  {
    path: '/dashboard',
    label: 'Dashboard',
    componentName: 'dashboard',
    nav: true,
  },
  {
    path: '/heroes',
    label: 'Heroes',
    componentName: 'heroes',
    nav: true,
  },
  {
    path: '/hero-detail/:id',
    label: 'HeroDetail',
    componentName: 'hero-detail',
    nav: false,
  },
];
```

続いて，`app.riot` を以下のように修正してください．

```diff
         <nav class="menu column">
-          <a href="/dashboard">Dashboard</a>
-          <a href="/heroes">Heroes</a>
+          <a
+            each={ page in state.pages }
+            if={ page.nav }
+            href={ page.path }
+          >
+            { page.label }
+          </a>
         </nav>
       <div>
-      <route path="/(dashboard)?">
-        <main is="dashboard" />
-      </route>
-      <route path="/heroes">
-        <main is="heroes" />
-      </route>
-      <route path="/hero-detail/:id">
-        <main is="hero-detail" id={ route.params.id } />
+      <route each={ page in state.pages } path={ page.path }>
+        <main is={ page.componentName } id={ route.params.id } />
       </route>
     </router>
```

これで各ルートの管理が少し楽になりました．

## active なコンポーネントの明示

現状ですと，ナビゲーションのどちらをクリックしたのかが画面からは判断がつきますが，ナビゲーション自体からは分かりにくいため，こちらを修正します（※ こちらは Angular のチュートリアルにはないものです）．`app.riot` を以下のように修正してください．

```diff
         <a
           each={ page in state.pages }
           if={ page.nav }
           href={ page.path }
+          class={ state.activePage === page ? 'active' : '' }
         >
           { page.label }
         </a>

// （中略）

   <script>
-    import { Router, Route} from '@riotjs/route'
+    import { Router, Route, route, toRegexp, match } from '@riotjs/route'

     export default {
       state: {
-        pages
+        pages,
+        activePage: null
+      },
+      onBeforeMount() {
+        const anyRouteStream = route('(.*)')
+        anyRouteStream.on.value((path) => {
+          const activePage = pages.find(p => match(path.pathname, toRegexp(p.path)))
+          this.update({
+            activePage
+          })
+        })
       },
```

ちょっといきなり見慣れないコードがたくさん出てきましたが，こちらは別のテンプレート [SPA (Webpack)](https://github.com/riot/webpack-spa-template/blob/c7ba166986d8d800d5836db574e884bf9efda03e/src/app.riot#L59-L72) から拝借しました．

要するに，**ルーティングのイベントが発火したのを検知し，そのパスを取得．`pages` 配列から一致するものを探す** という処理をしています．

一致したもので `state.activePage` を更新し，`<nav>` の当該 `<a>` の class 属性に `active` を付与しています．

:::details ほんの少しだけ解説

`@riotjs/route` が export している関数屋プロパティは他にもいくつかありまして，その中の３つ [route](https://github.com/GianlucaGuarini/rawth/blob/70cc9605a88d9515f3dc92527abf927fed8942af/index.next.js#L155), [toRegexp](https://github.com/GianlucaGuarini/rawth/blob/70cc9605a88d9515f3dc92527abf927fed8942af/index.next.js#L94), [match](https://github.com/GianlucaGuarini/rawth/blob/70cc9605a88d9515f3dc92527abf927fed8942af/index.next.js#L133) を利用していますが，これらはリンク先を見ても分かるように他のライブラリ [rawth](https://github.com/GianlucaGuarini/rawth) が export しているものになります．それぞれのメソッドの実装を見ますと

- `route`: 新しいイベントストリームを作成．指定されたパスに関するルーティングのイベントを検知する．イベント検知のコアロジックは [erre](https://github.com/GianlucaGuarini/erre) というライブラリで，[RxJS](http://reactivex.io/rxjs/) 等にインスパイアされている模様
- `toRegExp`: 第１引数に渡したパス（文字列）を正規表現に置換する（例：`/dashboard` -> `/^\/dashboard[\/#\?]?$/i`）．第２引数は正規表現のオプションの設定．詳しくは[こちら](/^/dashboard[/#?]?$/i)
- `match`: 第１引数にターゲットのパス（文字列），第２引数に正規表現の文字列を渡し，JavaScript 標準の `RegExp` インスタンスの [test](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/RegExp/test) メソッドを実行する．正規表現と指定された文字列を照合するための検索する

と言った感じです．余談ですが，上記のライブラリのソースコードを見てみると，エラーハンドリングのメソッドに `panic` という名前のメソッドが使われており，コアコミッターの方が `Golang` を使っているのかな，と推察できますね．
:::

## Not Found（404） ページを表示

http://localhost:3000/hoge という URL にアクセスすると，「そんなパスはないよー」というエラーが発生すると思います．一般的には存在しないパスにアクセスした場合は，Not Found（404）というページを表示しますので，ここでも `not-found` というコンポーネントを用意し，表示させましょう．

- `src/components/not-found/` ディレクトリ
- `src/components/not-found/not-found.riot` ファイル
- `src/components/not-found/not-found.spec.js` ファイル

を作成後，`not-found.riot` に以下を記述してください．今回はもうスタイルも一緒に実装しちゃいます．

```html
<not-found>
  <h1>Page not found</h1>
  <a href="/">go back</a>

  <style>
    a {
      padding: 1rem;
      text-decoration: none;
      margin-top: 10px;
      display: inline-block;
      background-color: #e8e8e8;
      color: #3d3d3d;
      border-radius: 4px;
    }
    a:hover {
      color: white;
      background-color: #42545c;
    }
  </style>
</not-found>
```

続いて，`app.riot` にて存在しないルートへのアクセスがあった場合は `not-found` コンポーネントを表示させます．

```diff
         </nav>
       </div>
-      <route each={ page in state.pages } path={ page.path }>
-        <main is={ page.componentName } id={ route.params.id } />
-      </route>
+      <div if={ state.showNotFound }>
+        <not-found />
+      </div>
+      <div if={ !state.showNotFound }>
+        <route each={ page in state.pages } path={ page.path }>
+          <main is={ page.componentName } id={ route.params.id } />
+        </route>
+      </div>
     </router>
+    <hr />
     <messages />

// （中略）

     import Heroes from "@components/heroes/heroes.riot";
     import Messages from '@components/messages/messages.riot';
     import HeroDetail from "@components/hero-detail/hero-detail.riot";
+    import NotFound from "@components/not-found/not-found.riot";
     import pages from "@/pages";

// （中略）

       state: {
         pages,
-        activePage: null
+        activePage: null,
+        showNotFound: false,
       },
       onBeforeMount() {
         const anyRouteStream = route('(.*)')
         anyRouteStream.on.value((path) => {
           const activePage = pages.find(p => match(path.pathname, toRegexp(p.path)))
           this.update({
-            activePage
+            activePage,
+            showNotFound: !activePage
           })
         })
       },
```

これで，`/hoge` など存在しないパスにアクセスすると，以下のように Not Found の文言が表示されます．

![Not Found ページの表示](/images/books/riotjs_toh/06_not_found.png)

以上で Chapter6「ナビゲーションの追加」は完了です．何かわからないことがあれば，遠慮なくコメントしてください！
