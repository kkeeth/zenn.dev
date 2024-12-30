---
title: 'Riot.js with Vitest でコンポーネントのユニットテスト'
emoji: '⚙️'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['riotjs', 'riot', 'vitest', 'ユニットテスト']
published: true
---

:::message
本記事は [Riot.js Advent Calendar 2024](https://qiita.com/advent-calendar/2024/riotjs) の第 19 日目(遅刻組) の記事になります！
:::

https://qiita.com/advent-calendar/2024/riotjs

こんにちは．株式会社カミナシで EM をしている [Keeth](https://twitter.com/kuwahara_jsri) こと桑原です．

[Riot.js（以下，riot）](https://riot.js.org/) の公式ボイラープレートの環境ではバンドラに [webpack](https://webpack.js.org/) が使われていますが，[昨日（18 日目）の記事]() で [Vite](https://vite.dev/) に変更しました．また，公式ボイラープレートで使われているテスティングフレームワークは JavaScript のテスティングフレームワークとしては最古参と言っても良い [Mocha](https://mochajs.org/) をベースに環境が用意されていますが，せっかく vite を用いているのであれば [Vitest](https://vitest.dev/) で性ということで，実際に書いてみました．^[ほぼほぼ vitest の紹介になります 🙇]参考になれば幸いです 💁 本記事内のコードはこちらのリポジトリになります．

https://github.com/kkeeth/riot-toh-demo/tree/with-vite

## Goodbye `Mocha`

では昨日と同様に，まずはプロジェクト内から mocha を削除します．

```Bash
$ pnpm rm mocha chai c8
```

## `Vitest` の導入

続いて vitest のインストールから．[vitest の公式ドキュメント](https://vitest.dev/guide/) に沿って進めましょう．

```Bash
$ pnpm add -D vitest
```

インストールが終わったら，公式ドキュメントのサンプルテストを試してみてもよいですが，大抵の JavaScript テスティングフレームワークのサンプルと同じのため，今回は割愛しますが，一点だけ，`package.json` の `scripts` を変更します 🙋

```diff
     "dev": "vite",
     "build": "vite build",
     "preview": "vite preview",
-    "test": "NODE_OPTIONS=\"--loader @riotjs/register\" c8 mocha --require jsdom-global/register src/**/*.spec.js",
+    "test": "vitest --reporter=verbose",
-    "cov": "c8 report --reporter=text-lcov",
-    "cov-html": "c8 report --reporter=html",
```

後ほど言及しますが，`test` コマンドの `--reporter=verbose` オプションを付けないと，各テストケースの詳細なケースの結果が表示されないため，明示的に付けています．他にも[様々な `reporter`](https://vitest.dev/guide/reporters.html) がありますので，好みでお選びください．

続いて，`vite.config.js` にテスト用の設定を追記します．

```diff
   },
+  test: {
+    environment: 'jsdom',
+  },
   build: {
```

環境設定として `test.environment` に `jsdom` を利用します．vitest は `Node.js` で実行されますが，`Node.js` には `document` オブジェクトが存在しません．しかし画面のテストにおいて `document` オブジェクトなどの Web 標準の API やオブジェクトにアクセスしたいケースはちょこちょこあります．そのために，[jsdom](https://www.npmjs.com/package/jsdom) を指定することでこれを解決します．

`jsdom` のドキュメントにもこのように書かれており，ブラウザ側の機能をエミュレートしてくれるライブラリであることがわかります．

> jsdom is a pure-JavaScript implementation of many web standards, notably the WHATWG DOM and HTML Standards, for use with Node.js. In general, the goal of the project is to emulate enough of a subset of a web browser to be useful for testing and scraping real-world web applications.

:::details vite と vitest で設定ファイルを分けたい場合
vitest 専用の設定ファイル `vitest.config.js` を分けることもできます．この場合はファイルを作成し，以下を追記します．

```js
import { defineConfig, mergeConfig } from 'vitest/config';
import viteConfig from './vite.config.js';

export default mergeConfig(
  viteConfig,
  defineConfig({
    test: {
      environment: 'jsdom',
    },
  }),
);
```

mergeConfig` 関数を用いることで，`vite`の環境設定を持ってきてマージすることで，プロジェクトでバンドルするときに一緒にトランスパイルしたりエイリアスだったり，裏で実行している処理をテスト時にもできます．今回は riot の拡張子`.riot`をトランスパイルして`.js` 変換しないと，vitest も「このファイルは何だ？」となりテストが実行できないので．

ただし，公式は一つのファイルで運用するのが推奨されています．

> However, we recommend using the same file for both Vite and Vitest, instead of creating two separate files.

:::

## 実践！

準備が整いましたので，いざテストを書いてみましょう 💁

最も簡単なケースとして，`Props（初期値）` を渡さずシンプルに文字列を表示する `Not Found` 用のコンポーネントのユニットテストを書いてみます．正しくレンダリングされているか？を確認するテストです．

**src/components/not-found/not-found.riot**

```html
<not-found>
  <h1>Page not found</h1>
  <a href="/">go back</a>

  <style>
    /** 省略 */
  </style>
</not-found>
```

**src/components/not-found/not-found.spec.js**

```js
import NotFound from './not-found.riot';
import { describe, expect, test } from 'vitest';
import { component } from 'riot';

describe('NotFound Unit Test', () => {
  const mountNotFound = component(NotFound);
  test('The component is properly rendered', () => {
    const div = document.createElement('div');
    const component = mountNotFound(div);
    expect(component.$('h1').innerHTML).to.be.equal('Page not found');
  });
});
```

テストで具体的にやっていることは，シンプルに「コンポーネントを仮想的にレンダリングし，`h1` タグの文言が期待値通り（`Page not found`）か？」というものです．

ここまで書けましたら実行します．

```bash
$ pnpm test

> riot-toh-demo@1.0.0 test /path/to/riot-toh-demo
> vitest --reporter=verbose


 DEV  v2.1.8 /path/to/riot-toh-demo

 ✓ src/components/not-found/not-found.spec.js (1)
   ✓ NotFound Unit Test (1)
     ✓ The component is properly rendered

 Test Files  1 passed (1)
      Tests  1 passed (1)
   Start at  23:22:03
   Duration  315ms (transform 19ms, setup 0ms, collect 25ms, tests 9ms, environment 155ms, prepare 30ms)

 PASS  Waiting for file changes...
       press h to show help, press q to quit
```

テストが無事に成功していることと，`watch` モードで実行されていることが確認できました！^[従来のテスティングフレームワークとの書き方にちかいので，サクッと導入できたので，このあたりの開発体験も考慮されているんだろうなと改めて感じます．]さらに，ここからは変更されたテストファイルのみが再実行されます．便利ですね〜

では続いて，`props` を渡すケースのテストも書いてみます．まずはテスト対象の `hero-detail` コンポーネント．

**src/components/hero-detail/hero-detail.riot**

```react
<hero-detail>
  <div if={ selectedHero.id }>
    <h2>{ selectedHero.name.toUpperCase() } Details</h2>
    <div>
      <span id="hero-id">id: { selectedHero.id }</span>
    </div>
    <div>
      <label for="hero-name">Hero name: </label>
      <input
        id="hero-name"
        type="text"
        value={ selectedHero.name }
        placeholder="name"
        oninput={ handleInput }
      />
    </div>
    <button type="button" onclick="{" goBack }>go back</button>
  </div>

  <script>
    import heroService from '@services/hero.service';

    export default {
      selectedHero: {},
      onBeforeMount(props) {
        heroService.on('getHero', (hero) => {
          this.selectedHero = hero;
        });

        heroService.getHero(Number(props.id));
      },
      handleInput(e) {
        this.selectedHero.name = e.target.value;
        this.update();
      },
      goBack() {
        history.back();
      },
    };
  </script>
  <style>
    /** 省略 */
  </style>
</hero-detail>
```

続いてモックデータ．

**src/services/mock-heroes.js**

```js
export const HEROES = [
  { id: 11, name: 'Dr Nice' },
  { id: 12, name: 'Narco' },
  { id: 13, name: 'Bombasto' },
  { id: 14, name: 'Celeritas' },
  { id: 15, name: 'Magneta' },
  { id: 16, name: 'RubberMan' },
  { id: 17, name: 'Dynama' },
  { id: 18, name: 'Dr IQ' },
  { id: 19, name: 'Magma' },
  { id: 20, name: 'Tornado' },
];
```

そして，本題のテストコード．

**src/components/hero-detail/hero-detail.spec.js**

```js
import HeroDetail from './hero-detail.riot';
import { describe, expect, test } from 'vitest';
import { component } from 'riot';
import { HEROES } from '@services/mock-heroes';

describe('HeroDetail Unit Test', () => {
  const mountMessages = component(HeroDetail);
  test('The component is properly rendered', () => {
    const div = document.createElement('div');
    const component = mountMessages(div, {
      id: HEROES[0].id,
    });
    expect(component.$('h2').innerHTML).to.be.equal(
      `${HEROES[0].name.toUpperCase()} Details`,
    );
    expect(component.$('#hero-id').innerHTML).to.be.equal(
      `id: ${HEROES[0].id}`,
    );
  });
});
```

`hero-detail` コンポーネントは，モックデータ（オブジェクト配列）の最初の要素の `id` パラメータを渡してレンダリングし，それをキーにコンポーネント内で対象のヒーロー情報を取得，計算処理しています．このコンポーネントに対し，今回はのテストではヒーローの名前，id が適切に HTML に反映されているか？を確認しています．

特に難しいこともなく，スムーズに書けましたし，実行も問題ありませんでした．実に簡単！^[本記事を書いてて，あまりにも入門すぎるなと感じてきています ←]

### 余談：カバレッジ

テストを書いたらカバレッジも出したくなるのがエンジニアの性です．そして，もちろん vitest にもカバレッジの出力は提供されており，[こちらのページ](https://vitest.dev/guide/coverage.html)にまとまっております 💁

:::details ここは完全に vitest を用いたカバレッジのお話で riot はほぼ関係ないので畳んでいます
カバレッジの出力は [v8](https://v8.dev/blog/javascript-code-coverage), [istanbul](https://istanbul.js.org/) の２種類が用意されており，デフォルトは前者の v8 だそうです．

今回は `istanbul` を採用します 💁

### カバレッジの出し方

まずは設定からいきましょう．

**vitest.config.js**

```diff
  test: {
    environment: 'jsdom',
+   coverage: {
+     provider: 'istanbul',
+     exclude: ['dist/**', 'src/mocks/**', 'public/**', 'index.js'],
+   },
  },
```

テスト対象としては余計なファイルも含まれているので除外しています．
続いて実行のための `scripts` を追加します．

**package.json**

```diff
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "test": "vitest --reporter=verbose",
+   "coverage": "vitest run --coverage",
    "prepublishOnly": "npm test"
  },
```

<br>
設定できましたら実行します 💁

```bash
$ pnpm coverage

> riot-toh-demo@1.0.0 coverage /path/to/riot-toh-demo
> vitest run --coverage

 RUN  v2.1.8 /path/to/riot-toh-demo
      Coverage enabled with istanbul

 ✓ src/components/dashboard/dashboard.spec.js (1)
 ✓ src/components/heroes/heroes.spec.js (1)
 ✓ src/components/messages/messages.spec.js (1)
 ✓ src/components/not-found/not-found.spec.js (1)
 ✓ src/components/hero-detail/hero-detail.spec.js (1)

 Test Files  5 passed (5)
      Tests  5 passed (5)
   Start at  01:30:21
   Duration  608ms (transform 297ms, setup 0ms, collect 665ms, tests 71ms, environment 1.04s, prepare 257ms)

 % Coverage report from istanbul
--------------------------------|---------|----------|---------|---------|-------------------
File                            | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s
--------------------------------|---------|----------|---------|---------|-------------------
All files                       |   81.35 |        0 |   77.77 |   81.03 |
 src                            |       0 |        0 |       0 |       0 |
  register-global-components.js |       0 |        0 |       0 |       0 | 3-15
 src/components/dashboard       |     100 |      100 |     100 |     100 |
  dashboard.spec.js             |     100 |      100 |     100 |     100 |
 src/components/hero-detail     |     100 |      100 |     100 |     100 |
  hero-detail.spec.js           |     100 |      100 |     100 |     100 |
 src/components/heroes          |     100 |      100 |     100 |     100 |
  heroes.spec.js                |     100 |      100 |     100 |     100 |
 src/components/messages        |     100 |      100 |     100 |     100 |
  messages.spec.js              |     100 |      100 |     100 |     100 |
 src/components/not-found       |     100 |      100 |     100 |     100 |
  not-found.spec.js             |     100 |      100 |     100 |     100 |
 src/services                   |   78.94 |      100 |      80 |   77.77 |
  hero.service.js               |   83.33 |      100 |     100 |   81.81 | 16,27
  message.service.js            |   66.66 |      100 |      50 |   66.66 | 10-11
  mock-heroes.js                |     100 |      100 |     100 |     100 |
--------------------------------|---------|----------|---------|---------|-------------------
```

良さげな感じです．現時点では全体の約 81%が網羅されており，`services` 周りが足りていないことも可視化されていますね 👍️

### レポーター

カバレッジの出力形式は，以下の 4 つを指定できますので，お好みのものを指定してください．デフォルトでは全てが指定された状態で，`text` 以外は `coverage` というディレクトリに出力されます．

- `text`: 標準出力
- `html`: html 形式で出力（`live-server` 等を用いてブラウザで閲覧できる）
- `json`: json 形式で出力（`coverage-final.json` というファイル名）
- `clover`: xml 形式で出力（`clover.xml` というファイル名）

```diff
    coverage: {
      provider: 'istanbul',
      exclude: ['dist/**', 'src/mocks/**', 'public/**', 'index.js'],
+     reporter: ['text', 'html'],
    },
```

ちなみに `html` だと以下のような出力になります．

![html でのカバレッジの出力結果](/images/vitest_coverage.png)

### test 時にもカバレッジを出力したい場合

私は分けたい派ですが，もしテスト実行時にも出力したい場合は `vite.config.js` に以下１行を追記してください．

```diff
  test: {
    environment: 'jsdom',
    coverage: {
+     enabled: true,
      provider: 'istanbul',
      exclude: ['dist/**', 'src/mocks/**', 'public/**', 'index.js'],
    },
```

:::

## 終わりに

以上，いかがでしたでしょうか．実に簡単に導入することができ，もう `vite + vitest` で良くね？感があります．^[新しいツール出るたびに同じこといっている気もします w] また `mock` や `spy` なども試してはいなく，非同期処理（`async`, `await` や `Promise` を用いたもの）のテストや，エラーケースのテストは書いていないので，それも追って試して追記しようと思います 💁

軽く触った感じはかなり体験がよく，少なくとも， `mocha + chai` を使う理由はまったくなく鳴ってしまったなと思います．今後 riot でプロジェクトを始めるときは `vite` を自分は使いますし，テストを書くときも同様に `vitest` 一択になるだろうと．もしまだ試していない方は是非お試しください！

ではでは(=ﾟ ω ﾟ)ﾉ
