---
title: 'React 用状態管理ライブラリ「Jotai」に入門'
emoji: '📖'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['react', 'jotai', 'state']
published: true
---

こんにちは．[株式会社ゆめみ](https://www.yumemi.co.jp/)の Keeth こと桑原です．

今回は [Jotai](https://github.com/pmndrs/jotai) という React 用の超軽量な状態管理ライブラリを触ってみたので勉強ログとしてまとめました．軽く使ってみた感触としては非常にシンプルで分かりやすく，導入も簡単でした．

ただ，既に `Jotai` リリース後ある程度時間が経っており，Google で検索していただくとわかるかと思いますが，Jotai に関する記事もいくつかありますので，二番煎じな内容もありますことをご了承頂ければと思います．

※一応自分が手を動かしたコードのリポジトリも載せておきます 👇

[https://github.com/kkeeth/my-jotai-demo](https://github.com/kkeeth/my-jotai-demo)

# TL;DR

- とにかく軽量で簡単
- Redux が冗長に感じ，もっとライトなものを探している人にオススメ
- 軽量な分ルールが少ないのでカオスにならないように注意

# `Jotai` の概要

触ってみると共感頂けるかと思いますが，まず感じたことは React の [`Context API`](https://ja.reactjs.org/docs/context.html#api) と [`Recoil`](https://recoiljs.org/) に非常に似ています．

ちなみに発音ですが，公式リポジトリにも以下のように記載がある通り，

> Jotai is pronounced "joe-tie" and means "state" in Japanese.

日本語の「状態」から名前と発音を取っていることがわかります 😂

# サイズ比較

私がミニマルなライブラリが好きなので，React 用状態管理ライブラリのサイズ比較をしてみたいと思います．対象は Redux, Mobx, Recoil, Zustand の名前を聞くようになったので，これらと Jotai のサイズを比べてみます．※いずれも minified + gzipped の数字です．

| name        | size（執筆時点）                                           | size（2025/01/13 時点）                                     |
| :---------- | :--------------------------------------------------------- | :---------------------------------------------------------- |
| redux       | [1.6kb](https://bundlephobia.com/result?p=redux@4.1.0)     | [1.4kb](https://bundlephobia.com/package/redux@5.0.1)       |
| react-redux | [5kb](https://bundlephobia.com/result?p=react-redux@7.2.4) | [4.1kb](https://bundlephobia.com/package/react-redux@9.2.0) |
| mobx        | [16.1kB](https://bundlephobia.com/result?p=mobx@6.3.0)     | [17.5kb](https://bundlephobia.com/package/mobx@6.13.5)      |
| recoil      | [19.7kb](https://bundlephobia.com/result?p=recoil@0.3.0)   | [23.5kb](https://bundlephobia.com/package/recoil@0.7.7)     |
| jotai       | [2.5kb](https://bundlephobia.com/result?p=jotai@0.16.4)    | [3.5kb](https://bundlephobia.com/package/jotai@2.11.0)      |
| zustand     | -                                                          | [588b](https://bundlephobia.com/package/zustand@5.0.3)      |

こう見ると，Zustand・Jotai は公式が謳っている通りかなり軽量であることがわかりますが，Redux は単体だと更に軽量なのが意外でした．逆に Recoil は思った以上にサイズが大きいことがわかりますね．

余談ですが，ダウンロード数比較をすると，やはり Redux が強いですね．ついで Mobx が頑張っている印象で，これはリリースが早かったというのも加味すると納得です．

![](/images/compare_jotai_with_others.png)

[https://npmtrends.com/jotai-vs-mobx-vs-react-redux-vs-recoil-vs-redux-vs-zustand](https://npmtrends.com/jotai-vs-mobx-vs-react-redux-vs-recoil-vs-redux-vs-zustand)

また，公式ドキュメントの[コンセプト](https://docs.pmnd.rs/jotai/basics/concepts)を読んだところ，Jotai が生まれた元々の理由というか考えは，React の余分な再レンダリングの問題を解決するためだったそうです．

開発者である Daishi Kato 氏がこのように述べているように，

> Jotai can be seen as a replacement for useState+useContext. Instead of creating multiple contexts, atoms share one big context.

`useState` と `useContext` の代わりとしての面もあるとのことです．

# `Jotai` の使い方

根本の考え方として，`Jotai` はローカルの React コンポーネントの state の値を `atoms` で置き換える．これを `useAtom` という `hook` に渡します．`useAtom hook` を使うことで `atom` の値を取得・更新をします．

では，具体的に見ていきましょう．

## プリミティブな `atom` を生成

まずは `atom` と呼ばれる最小単にの状態を保持したオブジェクトを生成し，管理します．このオブジェクトを生成するには，以下のように `atom` というファクトリ関数に初期値を渡して実行することで可能です．

```javascript
import { atom } from 'jotai';

const count = atom(0);
```

`atom` はその性質を考えるとコンポーネントで定義するのではなく，以下のように別途 `Atoms.js` のようなファイルに集約して一元管理し，コンポーネント毎に使いたい atom をインポートできるようにすることが望ましいですね．

**Atoms.js**

```javascript
import { atom } from 'jotai';

export const countAtom = atom(0);
export const nameAtom = atom('keeth');
```

**Hoge.js**

```js
import { countAtom, nameAtom } from './Atoms'

const Hoge = () => {
  return (/**/)
}
export default Hoge
```

> The atom value is stored in a Provider state.

とあるように，atom の値は内部的には `Provider`(`useContext` の Provider のラッパー) 内で使われている `useState` で保持されるようです．

さらに，atom メソッドの定義（TypeScript で書かれています）を見ますと，独自にセッター/ゲッターを定義できるようで，汎用的な実装になっていますね．これはありがたい．

```ts
// primitive atom
function atom<Value>(initialValue: Value): PrimitiveAtom<Value>;

// read-only atom
function atom<Value>(
  read: (get: Getter) => Value | Promise<Value>,
): Atom<Value>;

// writable derived atom
function atom<Value, Update>(
  read: (get: Getter) => Value | Promise<Value>,
  write: (get: Getter, set: Setter, update: Update) => void | Promise<void>,
): WritableAtom<Value, Update>;

// write-only derived atom
function atom<Value, Update>(
  read: Value,
  write: (get: Getter, set: Setter, update: Update) => void | Promise<void>,
): WritableAtom<Value, Update>;
```

## `useAtom` で `atom` を利用

`Atoms.js` で定義して `Hoge` コンポーネントで読み込んだので，実際のコンポーネントで利用するには，`useAtom` という関数の引数にわたす必要があります．

**Hoge.js**

```js
import { Provider, useAtom } from 'jotai';
import { countAtom } from './Atoms';

const Hoge = () => {
  // このコンポーネントでは countAtom を利用する
  // 合わせてセッターも返される
  const [count, setCount] = useAtom(countAtom);

  const handlePlus = () => setCount((value) => value + 1);
  const handleMinus = () => setCount((value) => value - 1);

  return (
    <>
      <h1>{count} </h1>
      <div className="buttons">
        <button onClick={handlePlus}>one up</button>
        <button onClick={handleMinus}>one down</button>
      </div>
    </>
  );
};
export default Hoge;
```

うん，分かりやすい．というか既視感があります（笑）この `useAtom` の使い方は `useState` と `useContext`　を合わせたように見えますね．

## `computed` - 計算済みの値

既存の atom に何かしらの計算処理をした新しい atom を生成するための `get` というメソッドが用意されています．例を見てみましょう．

**Atoms.js**

```js
import { atom } from 'jotai';

export const countAtom = atom(0);
export const nameAtom = atom('keeth');
// 大文字変換した名前用の atom
export const upperNameAtom = atom((get) => get(nameAtom).toUpperCase());
```

利用の仕方も同様で，使いたいコンポーネント側で `useAtom` の引数に渡してください．

「プリミティブな atom を生成」のところで転載した atom の定義を見ますと，`Promise<Value>` があるように，非同期処理をすることも考慮されているようですね！公式 Demo アプリでも使われていたのでコードを一部抜粋します．（デモのリンクは本記事末に記載しています）

```js
const postData = atom(async (get) => {
  const id = get(postId);
  const response = await fetch(
    `https://hacker-news.firebaseio.com/v0/item/${id}.json`,
  );
  return await response.json();
});
```

## `Provider` で初期値を与える

Redux の Provider と同様に，Jotai にも Provider があり，その下のコンポーネント全体でアクセスできる atom を設定することができます．まずは定義から．

```js
const Provider: React.FC<{
  initialValues?: Iterable<readonly [AnyAtom, unknown]>
  scope?: Scope
}>
```

`initialValues` というアトリビュートで設定すれば良さそうですね．Iterable な値にすることに注意．

```jsx
const countAtom = atom(0)

const Root = () => (
  <Provider initialValues={[countAtom, 1]]}>
    <Component />
  </Provider>
)
```

# その他のライブラリとの違い

サイズ比較で名前を上げた２つ（react-redux は redux と同じなので無視）との違いを軽く書き残します．

※ここはまだ入門したばかりなので，今後更新したいと思います！（するとは限らない）

## ▼ `Redux` との違い

Redux にあった以下の４つの機能・仕様が Jotai にはありません．

- `Actions` がない
- `Reducers` がない
- `Dispatchers` がない
- `Stores` がない

それにより，ライブラリ本体のサイズも小さくなりますし，開発するときのコードの記述量も減ります．その分，ルールがないとみんなが思い思いの atom を作ったり，`useState` で良いところを atom を生成してしまったりなど，カオスになる可能性もあるかなと思います 🤔

## ▼ `Recoil` との違い

公式リポジトリの README に以下のように記載がありました．

> **How does Jotai differ from Recoil?**
> ・Minimalistic API
> ・No string keys
> ・TypeScript oriented

驚いたのは，Recoil のソースを軽く見たところ，確かに TypeScript で書かれていないですね．package.json には `typescript` の記述があり，かつ test 実行時に使っているようですが，なぜ開発時に使っていないのかは謎です．

## 終わりに

初期リリースが[2020 年 8 月 30 日](https://github.com/pmndrs/jotai/releases/tag/v0.1.0)ですが，約一年後の今更知った情弱な自分です…😅

また公式サイトに行くと，色んなレシピもあり，他の状態管理ライブラリとの共存のサンプルコードもあったりと学びがあるので，本格的に導入して開発をしたい方はぜひドキュメントを見に行ってみてください．

とりま，自分は今後 React/Next で開発する際に状態管理ライブラリを入れたい場合は，Jotai は真面目に検討材料に入れていきたいなと思いました．参考になれば幸いです．

ではでは．

## 参考

- [公式リポジトリ](https://github.com/pmndrs/jotai)
- [Redux-Free State Management with Jotai](https://blog.bitsrc.io/redux-free-state-management-with-jotai-2c8f34a6a4a)
- [Jotai: 状態?! 日本語っぽい名称の React 用ステート管理ライブラリ](https://qiita.com/daishi/items/15f54c583a6d73891928)
- [How is jotai different from zustand?](https://github.com/pmndrs/jotai/issues/13)
- [Introducing Jotai || React State Manager Tutorial](https://www.youtube.com/watch?v=x9usS4l1VD0&feature=youtu.be) （YouTube 動画です）
- [Jotai Tutorial - Better than Recoil?](https://www.youtube.com/watch?v=TlP2XB6TFVw) （YouTube 動画です）
- [公式の Demo1](https://codesandbox.io/s/jotai-demo-47wvh)
- [公式の Demo2](https://codesandbox.io/s/jotai-demo-forked-x2g5d)
