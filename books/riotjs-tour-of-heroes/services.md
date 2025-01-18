---
title: 'Chapter5 データ取得用のサービスの作成'
---

今回は Angular フレームワークに備わっている Service という機能に相当するコンポーネントと，メッセージを表示するコンポーネントを作成します．

では今回もやっていきましょう！

# `Hero` サービスの追加

現在の実装では，`heroes` コンポーネント内で直接ヒーローデータを取得し保持していますが，コンポーネント内で直接データの取得や保存を行うべきではありません．コンポーネントはデータの受け渡しや表示に専念し，その他の処理は別ファイルへと切り出す方が望ましいです．

`Angular` チュートリアルの方針に沿って，アプリケーション全体でヒーローデータを取得できる `Hero` サービスを作成していきたいと思います．

## `Hero` サービスの作成

ということで，まずは以下のディレクトリとファイルを作成してきます．

- `src/services` ディレクトリ
- `src/services/hero.service.js` ファイル

ファイルができましたら中身を書いていきましょう．

## ヒーローデータの取得

では，現在 `heroes` コンポーネントで取得しているヒーローデータを `hero.service` で取得するように変更します．

まず `mock-heroes.js` ファイルを `src/services` ディレクトリに移動させ，`hero.service.js` ファイルにて `HEROES` 配列を読み込み，コール元にレスポンスするメソッド `getHeroes()` を定義します．

```js
import { HEROES } from '@services/heroes/mock-heroes';

export const getHeroes = () => {
  return HEROES;
};
```

次に，`heroes.riot` にて先程定義したメソッドをマウント前に処理する `onBeforeMount()` メソッドでコールし，`heroes` 変数の初期値としてセットします．

```diff

  <script>
-   import { HEROES } from './mock-heroes';
+   import { getHeroes } from '@services/hero.service';
    import HeroDetail from '@components/hero-detail/hero-detail.riot';

    export default {
      // 厳密にはこの記述はなくても問題ない
-     heroes: HEROES,
+     heroes: [],
      selectedHero: {},
+     onBeforeMount() {
+       this.heroes = getHeroes();
+     },
      handleInput(e) {
```

## 非同期処理の実装と Observable でイベントハンドリング

これで，ヒーローデータ取得のロジックを `heroes` コンポーネントから切り出せました．ただ，現状ですと現在はモックデータということもありますが，同期的にデータを取得しています．しかし実際のアプリケーションでは，外部 API をコールする事が多いでしょう．その場合，ブラウザがブロックされてしまわないように非同期処理を実装する必要があります．

ここから，`Angular` の ToH では非同期にデータをフェッチするために [RxJS](https://rxjs.dev/) というライブラリの `Observable` クラスを利用しますが，riot では [riot/observable](https://github.com/riot/observable) を利用しつつ，非同期処理を実装します．

:::message
1️⃣ version 3 以前を使われている方は，`observable` はコアモジュールのため，以下のように本体からインポートが可能です．

```js
import { observable } from 'riot';
```

2️⃣ `riot/observable` はイベントの送受信用のライブラリであり，非同期処理を内包しているわけではないため，非同期処理は別途書く必要があります．
:::

:::details riot コンポーネント内の非同期処理

riot コンポーネント内の組み込みメソッドでも，非同期処理を書くことはできます．以下例．

```js
export default {
  // レンダリング前に必ず実行
  async onBeforeMount() {
    const response = await fetch(/** URL */);
    const data = response.json();
  },
  // 非同期の関数を定義しておき，適宜呼び出したいところで呼び出す
  async myMethod(v) {
    const response = await fetch(/** URL */);
    const data = response.json();
  },
};
```

:::

まずは `riot/observable` をインストールしましょう．

```bash
npm install -S  @riotjs/observable
```

続いて，`riot/observable` を用いて `hero.service.js` を書き直しましょう．

```diff
  import { HEROES } from "@components/heroes/mock-heroes";
+ import observable from '@riotjs/observable'

- export const getHeroes = () => {
-   return HEROES;
- };
+ const heroService = {
+   async getHeroes() {
+     try {
+       // 実際は以下のように実装するが，今回はモックで実装
+       // const response = await fetch('https://api.+xample.com/heroes');
+       // const heroes = await response.json();
+       const heroes = HEROES;
+       this.trigger('heroesUpdated', heroes)
+     } catch (error) {
+       console.error('Failed to fetch heroes:', error);
+     }
+   }
+ };

+ observable(heroService);

+ export default heroService;
```

riot には非同期処理の機能や API がないため，愚直に実装するしかなく，`fetch API` を async-await で書くか，何かしらのフェッチャーライブラリ（[axios](https://axios-http.com/)，[ky](https://github.com/sindresorhus/ky) など）を使うことになるでしょう．

コードの記述的には前後しますが，`observable` メソッドを用いて，`heroService` オブジェクトに [Observer 機能](https://ja.wikipedia.org/wiki/Observer_%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3) を付与し，イベントのトリガーおよび監視を可能にします．これにより，

- `this.on`: イベントの監視および，callback の実行
- `this.one`: イベントの監視および，callback を一度だけ実行
- `this.off`: 監視を停止または，callback の削除
- `this.trigger`: イベントの発火

という `@riotjs/observable` の機能を使えるようになります．詳しくは [ドキュメント](https://github.com/riot/observable/blob/main/doc/README.md)　をご参照ください．

データを取得後，そのデータを引数に `this.trigger` メソッドを実行し，`heroesUpdated` イベントを発火させます．

続いて，`heroes.riot` も更新し，`heroesUpdated` イベントを監視し，コールバックを実行します．

```diff
 <script>
-   import { getHeroes } from '@/services/hero.service';
+   import heroService from '@/services/hero.service';
    import HeroDetail from '@components/hero-detail/hero-detail.riot';

    export default {
      heroes: [],
      selectedHero: {},
      onBeforeMount() {
-       this.heroes = getHeroes();
+       heroService.on('heroesUpdated', (heroes) => {
+         this.heroes = heroes;
+       });
+       heroService.getHeroes();
      },
      handleInput(e) {
```

`heroService` オブジェクトの `getHeroes` メソッド内の `this.trigger` で指定した引数 `heroes` を，`heroes.riot` コンポーネントの `heroService.on` のコールバック関数の引数で受け取れます．この値を用いてヒーローリスト（`this.heroes`）を更新しています．

:::message
`@riotjs/observable` の中身は， JavaScript の Map を用いたイベント名と，イベントハッカのタイミングをハンドリングする構造になっておりますので，`on` メソッドの定義前に `trigger` メソッドを発火してしまうと，期待した処理ができないことに注意してください．
:::

以上で画面を確認してみましょう．挙動としては全く変わらないと思いますが，API コール処理をサービスオブジェクトに切り出せました．

# `Message` サービスの追加

続いて，メッセージを表示するためのコンポーネントとサービスを実装します．

## `Message` コンポーネントとサービスの作成

まずは必要なフォルダとファイルを作ります．

- `src/components/messages/`
- `src/components/messages/messages.riot`
- `src/components/messages/messages.spec.js`
- `src/services/messages.service.js`

作成できましたら，`.spec.js` ファイルは後回しにして，`.riot` ファイルの方に以下を追記してください．中身は仮です．

```html
<messages>
  <h2>Messages</h2>
</messages>
```

そうしましたら，これを読み込んで画面に表示させるために `app.riot` でインポートし，配置しましょう．

```diff
   <div class="container">
     <h1>{ props.title }</h1>
     <heroes />
+    <messages />
   </div>

   <script>
     import Heroes from "@components/heroes/heroes.riot";
+    import Messages from '@components/messages/messages.riot';
   </script>
```

続いて，`messages.service.js` を `heroes.riot` にてインポートします．

```diff
  <script>
    import heroService from '@/services/hero.service';
+   import messageService from '@/services/message.service';
    import HeroDetail from '@components/hero-detail/hero-detail.riot';
    import Messages from '@components/messages/messages.riot';
```

以上で，準備は完了です．

## `Message` サービスの処理の実装

ではここからは，具体的に処理を実装します．まずは message サービスからです．

```js
import observable from '@riotjs/observable';

const messageService = {
  messages: [],
  add(message) {
    this.messages.push(message);
    this.trigger('messagesAdded', this.messages);
  },
};

observable(messageService);

export default messageService;
```

単純に追加のための `add` メソッドを追加し，受け取ったメッセージを配列に格納しています．その後，`this.trigger` メソッドで更新後の配列を引数に `messagesAdded` イベントを発火させ，メッセージを追加，受け取ったコンポーネントで処理を委譲します．

また，ヒーローデータを取得した際のメッセージを追加したいと思います．`hero.service.js` に以下を追記してください．

```diff
+ import messageService from "@/services/message.service";

  const heroService = {
    async getHeroes() {
      //（省略）
      const heroes = HEROES;
+     messageService.add('HeroService: fetched heroes');
      this.trigger('heroesUpdated', heroes)
    } catch (error) {
```

さらに `heroes.riot` で，ヒーロー選択時のイベント発火のメッセージを追加する処理を書きましょう．

```diff
  handleSelect(e) {
    this.selectedHero.id = e.target.closest('button').data.id;
    this.selectedHero.name = e.target.closest('button').data.name;
+   messageService.add(`HeroesComponent: Selected hero id=${this.selectedHero.id}`);
    this.update();
  }
```

処理が書けたら，画面に表示をしたいので，`messages.riot` を修正します．

```diff
  <messages>
    <h2>Messages</h2>
+
+   <div each={ message in messages }>{ message }</div>
+
+   <script>
+     import messageService from '@services/message.service';
+
+     export default {
+      messages: [],
+      onBeforeMount() {
+        messageService.on('messagesAdded', (messages) => {
+          this.messages = messages;
+          this.update();
+        });
+      }
+     }
+   </script>
  </messages>
```

`messagesService` をインポートし，`messagesAdded` イベントを監視．送られたメッセージ配列を受け取るようにしています．後は，それを `each` で表示しています．シンプルですね．

ここまで実装できますと，以下のように，ヒーローを選択するたびにメッセージが表示されるようになります．

![メッセージリストの表示](/images/books/riotjs_toh/05_show_messages.png)

## メッセージ一覧のクリア

続いて，ボタンをクリックすると表示しているメッセージの一覧をクリアする処理を書いていきます．まずは，`message.service.js` に `clear` メソッドを追加します．

```diff
  add(message) {
    this.messages.push(message);
    this.trigger('messagesAdded', this.messages)
- },
+ },
+ clear() {
+   this.messages = [];
+   this.trigger('messagesCleared', this.messages)
+ }

```

続いて `messages.riot` を以下のように変更してください．

```diff
  <h2>Messages</h2>

  <div each={ message in messages }>{ message }</div>
+ <button
+   type="button"
+   class="clear"
+   onclick={ clearMessages }
+ >Clear messages</button>

// （中略）

  onBeforeMount() {
    messageService.on('messagesAdded', (messages) => {
      this.messages = messages;
    });
+   messageService.on('messagesCleared', (messages) => {
+     this.messages = messages;
+     this.update();
+   });
- }
+ },
+ clearMessages() {
+   messageService.clear();
+ }
```

:::details 初期レンダリング時のメッセージ表示
現状は初期レンダリング時に `HeroService: fetched heroes` のメッセージが表示されず，何かヒーローを選択した際にメッセージ一覧に表示されると思います．

これは riot のネストしたコンポーネントのレンダリングの都合になってしまいますが，riot は子コンポーネントのマウントが先に走り，その後に`heroes` コンポーネントのマウント処理が走ります．

そのため，上記の実装ですとまだ `messages` コンポーネントが先にレンダリングされてしまい，`messageService.trigger` メソッドが後から走るため，イベントの発火も後回しとなります．ちなみに `messages` コンポーネントの `this.messages` 変数を確認してみると，

```js
onMounted() {
  this.update();
  console.log(this.messages);
},
```

上記は空配列となります．また逆に，`heroes.riot` で `onMounted` メソッドで `messageService` の `messages` 変数を確認しますと，メッセージが格納されていることが確認できます．

```js
// heroes.riot
onMounted() {
  console.log(messageService.getMessages());
},

// message.service.js
getMessages() {
  return this.messages;
}
```

ですので，初期レンダリング時にメッセージを表示したい場合は，不本意ですが，`messages.riot` コンポーネントで

```diff
  onBeforeMount() {
    messageService.on('messagesAdded', (messages) => {
      this.messages = messages;
      this.update();
    });
    messageService.on('messagesCleared', (messages) => {
      this.messages = messages;
      this.update();
    });

+   messageService.add('HeroService: fetched heroes');
  },
```

と発火させれば Angular のチュートリアルと同じ挙動となります．
※他に方法があれば教えていただけると助かります！🙇‍♂
:::

最後にスタイリングしましょう 🙋‍♂

```diff
   </script>
+ <style>
+   /* MessagesComponent's private CSS styles */
+   h2 {
+     color: #A80000;
+     font-family: Arial, Helvetica, sans-serif;
+     font-weight: lighter;
+   }
+   .clear {
+     color: #333;
+     background-color: #eee;
+     margin-bottom: 12px;
+     padding: 1rem;
+     border-radius: 4px;
+     font-size: 1rem;
+   }
+   .clear:hover {
+     color: white;
+     background-color: #42545C;
+   }
+   </style>
  </messages>
```

では，動作を確認してみましょう．

![メッセージのクリア](/images/books/riotjs_toh/05_clear_messages.gif)

以上，Chapter5「データ取得用のサービスの作成」の実装完了です！何かわからないことがあれば，遠慮なくコメントしてください！
