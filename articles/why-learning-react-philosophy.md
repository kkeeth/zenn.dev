---
title: 'なぜ React 哲学を守らないといけないのか？'
emoji: '🎃'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['react', '哲学', '思想', '純関数']
published: true
---

こんにちは．年々病気に弱くなってきて体力の低下と年齡を感じる今日このごろです． 株式会社カミナシで EM をしております，[Keeth](https://x.com/kuwahara_jsri) こと桑原です．

毎日お世話になっている 𝕏 で以下のポストを見かけました．

https://x.com/le_panda_noir/status/1962638427628937481

こちらの投稿を見たとき「これは読みたい！」と思ったのですが，しっかりリプライで

> 書きたいってか、こういう感じのを誰か書いて使わせてくれ

と書かれていました（笑）ただ，自分でもこれは整理してみたくなったので，色々調べつつ自分なりに思うところを書いてみます．むっちゃ遅くなってしもた．

※あまり React の思想については強くないので，間違っている記述があればバンバンコメントいただけると嬉しいです 🙇‍♂️

---

## なぜ React 哲学を守らないといけないのか？

React は良くも悪くも，とても表現力が高い柔軟なライブラリで，哲学通りじゃない柔軟な書き方が割とできたりします．上記の投稿でクロパンダさん御本人も書かれているように，哲学に従っていないコードは，React の将来の変更等（破壊的変更も含む）で互換性を失い，アプリケーションで利用している React のバージョンアップにより壊れる可能性があります．

その中で哲学を守ることは，アプリケーションを**予測可能で，バグりにくく，将来の React の進化に乗り遅れない，そして変更に耐えうるようにするため**という面が大きいと思っています．

React は，UI を `「宣言的」` に（つまり「どうやるか」ではなく「どうあるべきか」を）記述するためのライブラリです．この根本に従うことで，コードはシンプルで理解しやすくなります．React 哲学はこの宣言的な UI 構築を支えるための「ルール」や「お作法」と言ってもよく，だからこそ守るべきだと自分は考えます．

## 具体的にバージョンアップで壊れる危険性のあるコード例

では具体的に，２つほど壊れる危険性のあるコードを考えたので見ていきましょう．

### ① Strict Mode で問題発生

典型的なのが，**クリーンアップ関数を持たない`useEffect`** です．

```tsx
type User = {
  id: string;
  name: string;
};

type UserProfileProps = {
  userId: string;
};

const UserProfile = ({ userId }: UserProfileProps) => {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    // APIを呼び出してカウンターを増やす副作用
    fetch(`/api/users/${userId}/increment-view-count`, { method: 'POST' });

    fetch(`/api/users/${userId}`)
      .then((res) => res.json())
      .then(setUser);
  }, [userId]);

  return <div>{user?.name}</div>;
};
```

このコードは一見問題なく見えますが，`Strict Mode` では React が意図的に `useEffect` を 2 回実行するため，ページビューが 2 回カウントされてしまいます．開発環境だけの問題と思われるかもしれませんが，これが将来 production でも実行されるようになれば，閲覧回数の数字が期待値とは異なる結果になってしまいます．

::::details ※一点補足
[𝕏 で補足いただきました](https://x.com/le_panda_noir/status/1992572743108448492?s=20)．`useEffect` が production で 2 回実行されるような未来はないと言ってよいでしょう．これはあくまで私の妄想ですが，少々過ぎるものでした．

公式ドキュメントの [<StrictMode>](https://ja.react.dev/reference/react/StrictMode#fixing-bugs-found-by-double-rendering-in-development) のページの「補足」の項でも太字で強調されておりました．

> Strict Mode は開発中に以下のチェックを有効にします：
>
> - コンポーネントは、純粋でない (impure) レンダーによって引き起こされるバグを見つけるために、レンダーを追加で 1 回行います。
> - コンポーネントは、エフェクトのクリーンアップし忘れによるバグを見つけるために、エフェクトの実行を追加で 1 回行います。
> - コンポーネントは、ref のクリーンアップし忘れによるバグを見つけるために、ref コールバックの実行を追加で 1 回行います。
> - コンポーネントが非推奨の API を使っていないかチェックします。
>
> **これらのチェックはすべて開発環境専用であり、本番用ビルドには影響しません。**

::::

さらに [`Activity`](https://ja.react.dev/reference/react/Activity) という機能を使っているときも，この問題は発生する可能性があります．

::::message
**Activity とは？**
ユーザーの目に見えない（CSS の `display: none`）裏側で，UI をあらかじめレンダリングしておく機能です．例えばタブを切り替えたときに，次の画面が一瞬で表示されるような体験を実現します．
::::

この機能が有効になると，React はパフォーマンス向上のために

1. コンポーネントを画面に表示する前に裏でレンダリング（= Effect を再作成）
2. その後，不要と判断すれば Effect を破棄（= クリーンアップを実行）
3. 再度表示する（= 再度 Effect を作成）

といった挙動をします。

先ほどの `UserProfile` コンポーネントがこの機能で使われると，**ユーザーが一度も見ていないのに，裏のレンダリングだけでページビューが記録されてしまう**，という可能性があります．

#### 正しいコード

`useEffect` には，その処理を「打ち消す」ためのクリーンアップ関数を常に意識します．

```diff tsx
 type User = {
   id: string;
   name: string;
 };

 type UserProfileProps = {
   userId: string;
 };

 const UserProfile = ({ userId }: UserProfileProps) => {
   const [user, setUser] = useState<User | null>(null);

   useEffect(() => {
+    let cancelled = false;
+
-    // APIを呼び出してカウンターを増やす副作用
-    fetch(`/api/users/${userId}/increment-view-count`, { method: 'POST' });
-
+    // 副作用のない読み取り専用操作
     fetch(`/api/users/${userId}`)
       .then(res => res.json())
-      .then(setUser);
+      .then((data: User) => {
+        if (!cancelled) {
+          setUser(data);
+        }
+      });
+
+    return () => {
+      cancelled = true;
+    };
   }, [userId]);

+  // カウントアップは別のユーザーアクション時に実行
+  const handleViewCount = () => {
+    fetch(`/api/users/${userId}/increment-view-count`, { method: 'POST' });
+  };
+
-  return <div>{user?.name}</div>;
+  return (
+    <div onClick={handleViewCount}>
+      {user?.name}
+    </div>
+  );
 }
```

このように，React の `useEffect` を正しく使うには，「一度きりの処理」をどう実現するかではなく， **「何度実行されても最終的な結果が正しくなるように処理を設計する」** という考え方が重要だなぁと思います．

### ② React Concurrent Features で問題発生

```tsx
let globalCounter = 0;

type Item = {
  id: string;
  name: string;
  processed: boolean;
};

type ImpureComponentProps = {
  items: Item[];
};

const ImpureComponent = ({ items }: ImpureComponentProps) => {
  // レンダリング中にグローバル変数を変更
  globalCounter++;

  // 外部の状態を直接変更
  items.forEach((item) => {
    item.processed = true;
  });

  return (
    <div>
      <p>Rendered {globalCounter} times</p>
      <ul>
        {items.map((item) => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </div>
  );
};
```

このコードでは，React 18 の Concurrent Rendering でレンダリングが中断・再開される際に，純粋なコンポーネントではないため予期しない動作を引き起こします．また，グローバル変数の値が予測不能になります．さらに，props を直接変更することで，親コンポーネントに予期しない影響を与えてしまいます．

#### 正しいコード

```diff tsx
-let globalCounter = 0;
-
 type Item = {
   id: string;
   name: string;
-  processed: boolean;
+  processed?: boolean;
 };

-type ImpureComponentProps = {
+type PureComponentProps = {
   items: Item[];
 };

-const ImpureComponent = ({ items }: ImpureComponentProps) => {
+const PureComponent = ({ items }: PureComponentProps) => {
-  // レンダリング中にグローバル変数を変更
-  globalCounter++;
-
-  // 外部の状態を直接変更
-  items.forEach(item => {
-    item.processed = true;
-  });
+  // 元の配列を変更せずに新しい配列を作成（純粋な計算）
+  const processedItems = items.map(item => ({
+    ...item,
+    processed: true
+  }));

+  // レンダリング回数の表示が必要な場合は、親コンポーネントで管理する
   return (
     <div>
-      <p>Rendered {globalCounter} times</p>
       <ul>
-        {items.map(item => (
-          <li key={item.id}>{item.name}</li>
+        {processedItems.map(item => (
+          <li key={item.id}>
+            {item.name} {item.processed ? '✓' : ''}
+          </li>
         ))}
       </ul>
     </div>
   );
 }
```

### ③ `useEffect` の `deps` に `useMemo` 等でメモ化した値を使う

React 公式ブログの「[React Compiler のアップグレード](https://ja.react.dev/blog/2024/12/05/react-compiler-beta-release#upgrade-guide)」という記事の中で，以下のような記述があります．

> ただし、プロダクトコードは JavaScript では必ずしも静的に検出できない形で React のルールに違反することがあるため、メモ化を変更することで予期しない結果が生じることがあります。例えば、メモ化済みの値がコンポーネントツリーのどこかで useEffect の依存値として使用されているかもしれません。このような値のメモ化の有無や挙動が変化することで、対応する useEffect の実行が過剰あるいは過少になる可能性があるのです。useEffect は同期のためだけに使用することを推奨していますが、あなたのコードベースには、特定の値の変更にのみ反応する必要があるエフェクトなど、他のユースケースのための useEffect が含まれているかもしれません。

書かれているとおり，**メモ化された値を `useEffect` の依存配列に入れると，メモ化の挙動変更によって Effect の実行タイミングが変わってしまう** 可能性があります．

```tsx
type SearchParams = {
  query: string;
  category: string;
};

type SearchResultsProps = {
  query: string;
  category: string;
};

type Result = {
  id: string;
  title: string;
};

const SearchResults = ({ query, category }: SearchResultsProps) => {
  const [results, setResults] = useState<Result[]>([]);

  // 検索パラメータをメモ化
  const searchParams = useMemo(() => ({ query, category }), [query, category]);

  useEffect(() => {
    // メモ化された値に依存してAPIを呼び出す
    fetch(`/api/search?q=${searchParams.query}&cat=${searchParams.category}`)
      .then((res) => res.json())
      .then(setResults);
  }, [searchParams]); // メモ化された値を依存配列に入れている

  return (
    <ul>
      {results.map((result) => (
        <li key={result.id}>{result.title}</li>
      ))}
    </ul>
  );
};
```

このコードでは，`searchParams` がメモ化されているため，`query` や `category` が変わらない限り同じオブジェクト参照を返します．しかし，将来的に React Compiler の最適化や内部実装の変更により：

- メモ化の挙動が変わる（再計算のタイミングが変わる）
- メモ化が自動的に追加/削除される
- メモ化の粒度が変わる

といった変更があったとしましょう．その結果，`useEffect` が期待通りに実行されなくなる（実行されすぎる，あるいは実行されない）リスクがあると上記の記述を読むと解釈できると思います．

#### 変更後のコード

メモ化された値ではなく，**元の依存値を直接依存配列に入れる** ことで，React の内部実装変更の影響を受けにくくなります．

```diff tsx
 type SearchParams = {
   query: string;
   category: string;
 };

 type SearchResultsProps = {
   query: string;
   category: string;
 };

 type Result = {
   id: string;
   title: string;
 };

 const SearchResults = ({ query, category }: SearchResultsProps) => {
   const [results, setResults] = useState<Result[]>([]);

-  // 検索パラメータをメモ化
-  const searchParams = useMemo(() => ({ query, category }), [query, category]);

   useEffect(() => {
-    // メモ化された値に依存してAPIを呼び出す
-    fetch(`/api/search?q=${searchParams.query}&cat=${searchParams.category}`)
+    // 元の値を直接使用
+    fetch(`/api/search?q=${query}&cat=${category}`)
       .then(res => res.json())
       .then(setResults);
-  }, [searchParams]); // メモ化された値を依存配列に入れている
+  }, [query, category]); // 元の依存値を直接指定

   return (
     <ul>
       {results.map(result => (
         <li key={result.id}>{result.title}</li>
       ))}
     </ul>
   );
 }
```

このように，`useEffect` の依存配列には **プリミティブな値や，直接的な依存関係を持つ値を入れる** ことが良いのかなと考えます．メモ化は主にレンダリングのパフォーマンス最適化のために使うべきで，Effect の実行制御のために使うべきではないと私は理解しています．

## React 哲学を守る = コンポーネントを「純粋」に保つ

これまでの記述から想像できるように，React 哲学の核となる考え方は公式ドキュメントにも書いてありますが，**[コンポーネントを純粋に保つ](https://ja.react.dev/learn/keeping-components-pure)** ことです．ドキュメント内で使われている言葉をお借りすると，[純関数 (pure function)](https://wikipedia.org/wiki/Pure_function) のように保つということです．

ドキュメントでは，コンポーネントの純粋性について以下のように述べられています．

> - コンポーネントは純粋である必要がある。すなわち：
>   - **コンポーネントは自分の仕事に集中する**。レンダー前に存在していたオブジェクトや変数を書き換えない。
>   - **入力が同じなら出力も同じ**。同じ入力に対しては、常に同じ JSX を返すようにする。
>
> * レンダーはいつでも起こる可能性があるため，コンポーネントは相互の呼び出し順に依存してはいけない
> * コンポーネントがレンダーに使用する入力値を書き換えない。これには props，state，コンテクストが含まれる。画面を更新するためには既存のオブジェクトを書き換えるのではなく，代わりに [state をセットする](https://ja.react.dev/learn/state-a-components-memory)。
> * コンポーネントのロジックはできるだけコンポーネントが返す JSX の中で表現する。何かを「変える」必要がある場合，通常はイベントハンドラで行う。最終手段として `useEffect` を使用する
>
> 以上，「まとめ」より抜粋

一言で言えば， 特殊なこと・テクニカルなことをせず **React 哲学に忠実にコードを書きましょう** と解釈しました．つまり，哲学に従えばコンポーネントの純粋性は担保できるなと．

また，`React はなぜ純粋性を重視するのか？ ` という欄に

> 純関数を書くことには訓練が必要だが，それにより React パラダイムの威力が発揮される

とあります．まさにそのとおりで，純関数を書くことは意外と難しい（自分も含む）．いや，React のコードを書けと言われれば書けはしますし，アプリケーションとして不具合なく動作させられますが，**ちゃんと純関数になっていない** なんてことがよくあります．

:::message
**よくある「純関数になっていない」例**

例えば以下のようなコードは，一見問題なく動作しますが純関数ではありません：

```tsx
// BAD：レンダリング中に外部の変数を変更している
let renderCount = 0;

type CounterProps = {
  count: number;
};

const Counter = ({ count }: CounterProps) => {
  renderCount++; // レンダリング中にコンポーネント外の変数を変更
  return (
    <div>
      Count: {count}, Rendered: {renderCount} times
    </div>
  );
};
```

```tsx
// BAD：propsを直接変更している
type Todo = {
  id: string;
  title: string;
  priority: number;
};

type TodoListProps = {
  todos: Todo[];
};

const TodoList = ({ todos }: TodoListProps) => {
  todos.sort((a, b) => a.priority - b.priority); // propsを直接変更
  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>{todo.title}</li>
      ))}
    </ul>
  );
};
```

```tsx
// BAD：レンダリング中にAPIを呼び出している
type User = {
  name: string;
};

type UserProfileProps = {
  userId: string;
};

const UserProfile = ({ userId }: UserProfileProps) => {
  // 型エラー：Promise<any> を User 型に代入できない
  const user: User = fetch(`/api/users/${userId}`).then((r) => r.json()); // レンダリング中に副作用
  return <div>{user?.name}</div>;
};
```

:::

何度も登場している，この「純粋性」こそが React 哲学の根幹ですし，そもそもドキュメントの冒頭に，

> React はこのような概念に基づいて設計されています。**React は、あなたが書くすべてのコンポーネントが純関数であると仮定しています**。 つまり、あなたが書く React コンポーネントは、与えられた入力が同じであれば、常に同じ JSX を返す必要があります。

と書かれており，純関数で書くことが前提となっております．

また，コンポーネントを「状態から UI への変換器」として捉えると，UI の状態管理の考え方がシンプルになり，バグの予測や再現が容易になるのではないかと，今は考えるようになってきました．逆に言えば，純粋性を破ったコンポーネントは「いつ，何が起こるか分からない」不確定な動作をする可能性があり，React が提供する最適化の恩恵も受けられなくなってしまうのだと思います．

:::details ぶっちゃけ
今回の話は **だいたい React の公式ドキュメント「[コンポーネントを純粋に保つ](https://ja.react.dev/learn/keeping-components-pure)」に書かれています．**

https://ja.react.dev/learn/keeping-components-pure

ある程度書いてからこのドキュメントと照らし合わせ，修正していったら，「あれ，この記事いらなくね？」と思うなどしましたが，途中まで書いたので自分の学習の意味でも書ききってしまおうかなと．インターネットを更に散らかしてすんません！

また，純粋性については，私よりもよっぽど知見のある uhyo さんの [**React コンポーネントが「純粋である」とはどういうことか？　丁寧な解説**](https://zenn.dev/uhyo/articles/react-pure-components) という記事がとても素晴らしいので，是非こちらを読まれることをおすすめします！

https://zenn.dev/uhyo/articles/react-pure-components
:::

## まとめ

以上，React 哲学を守ることは，単に「ルールだから」という形式的なものではなく，React というライブラリの恩恵を最大限に受け，将来の変化にも耐えうる，堅牢で予測可能な UI を構築するための最も合理的な方法なのだと理解しました．

ではでは 👋
