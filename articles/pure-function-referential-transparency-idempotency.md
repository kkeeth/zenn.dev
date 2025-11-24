---
title: '純関数・参照透過性・冪等性の違いが分からなくなったのでまとめる'
emoji: '💡'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['関数型プログラミング', '純関数', '参照透過性', '冪等性']
published: false
---

こんにちは．最近関数型プログラミングの概念をちゃんと理解したい欲が出てきました．株式会社カミナシで EM をしております，[Keeth](https://x.com/kuwahara_jsri) こと桑原です．

以前 [「なぜ React 哲学を守らないといけないのか？」](https://zenn.dev/keeth/articles/why-learning-react-philosophy) という記事を書きましたが，執筆中に純関数の定義に触れたときふっと頭に浮かんだ疑問が，「これって冪等性のこと？いや，参照透過性？というか，そもそも自分がこの２つの言葉の概念や定義が曖昧かも…」ということで，明確に違いを理解したいため，整理してみます．

※もし間違っている記述があればバンバンコメントいただけると嬉しいです 🙇‍♂️

---

## TL;DR

- 純関数・参照透過性・冪等性は似ているようで違う概念
- 「冪等だけど純関数ではない」という例がある（DB 更新など）
- 関数型プログラミングでは純関数と参照透過性，API 設計では冪等性が重要
- React では純関数の概念をコンポーネントに，副作用の処理では冪等性を意識する

## 純関数（Pure Function）とは？

まずは純関数から．これは関数型プログラミングの基礎中の基礎ですね．

純関数は以下の２つの条件を満たす関数です：

1. **同じ入力**（引数）を与えられたとき、常に**同じ出力**（戻り値）を返す。（決定論的である）
2. 関数の実行中に**外部の状態を変更**したり（副作用），**外部の状態に依存**したりしない。

### 純関数の例

```typescript
// 純関数
const add = (a: number, b: number): number => a + b;

add(1, 2); // 3
add(1, 2); // 3（何度呼んでも同じ結果）
```

```typescript
// 純関数
const double = (arr: number[]): number[] => arr.map((x) => x * 2);

const nums = [1, 2, 3];
double(nums); // [2, 4, 6]
console.log(nums); // [1, 2, 3]（元の配列は変更されていない）
```

### 純関数じゃない例

```typescript
// 純関数じゃない（外部の状態に依存）
let multiplier = 2;
const multiply = (x: number): number => x * multiplier;

multiply(5); // 10
multiplier = 3;
multiply(5); // 15（同じ入力なのに結果が違う）
```

```typescript
// 純関数じゃない（副作用がある）
let counter = 0;
const increment = (): number => {
  counter++; // グローバル変数を変更
  return counter;
};

increment(); // 1
increment(); // 2（同じ入力（引数なし）なのに結果が違う）
```

```typescript
// 純関数じゃない（外部環境や実行時刻に依存）

console.log(Date.now()); // 実行時刻に依存
console.log(Math.random()); // 毎回ランダムな値
```

## 参照透過性（Referential Transparency）とは？

参照透過性は，プログラミングにおける式の性質を表す概念です．

**「式をその評価結果で置き換えても，プログラムの動作が変わらない」** という性質を指します．

### 参照透過な例

```typescript
const add = (a: number, b: number): number => a + b;
const multiply = (x: number, y: number): number => x * y;

// この式は
const result = multiply(add(2, 3), 4);

// こう置き換えても同じ結果になる
const result = multiply(5, 4); // add(2, 3) を 5 に置き換えた

// さらにこう置き換えても同じ
const result = 20;
```

この例では，`add(2, 3)` という式を評価結果の `5` に置き換えても，プログラムの動作は変わりません．これが参照透過性です．

### 参照透過じゃない例

```typescript
let count = 0;
const incrementAndGet = (): number => {
  count++;
  return count;
};

// この式は
const x = incrementAndGet() + incrementAndGet();

// こう置き換えると動作が変わってしまう
const x = 1 + 2; // 期待: 3, しかし実際は異なる可能性がある
```

`incrementAndGet()` は呼び出すたびに異なる値を返すので，式を評価結果で単純に置き換えることができません．つまり参照透過ではありません．

### 純関数と参照透過性の関係

実は，**純関数であれば参照透過性を持ちます**．逆に言えば，**参照透過ではない関数は純関数ではありません**．

ただし，概念としては：

- **純関数**: 関数の性質
- **参照透過性**: 式の性質

と，着目点が微妙に違うんですね 🤔

## 冪等性（Idempotency）とは？

冪等性は，**同じ操作を何度実行しても，1 回実行した時と同じ結果になる** という性質です．

ここが重要なのですが，冪等性は **「2 回目以降の操作は，1 回目が作った状態に対して，何の影響も与えない」** という点です．

### 冪等な例

```typescript
// 冪等な関数
const abs = (x: number): number => Math.abs(x);

abs(-5); // 5
abs(-5); // 5
abs(-5); // 5（何度実行しても同じ）
```

```typescript
// 冪等な操作（データベース更新）
const updateUserName = (userId: number, name: string): void => {
  // ユーザーの名前を更新
  db.users.update({ id: userId }, { name: name });
};

updateUserName(1, 'Keeth');
// 1回目: { id: 1, name: 'Taro' } → { id: 1, name: 'Keeth' }（状態が変わっている）
updateUserName(1, 'Keeth');
// 2回目: { id: 1, name: 'Keeth' } → { id: 1, name: 'Keeth' }（結果は同じ）
```

### 冪等じゃない例

```typescript
// 冪等じゃない
let counter = 0;
const increment = (): number => {
  counter++;
  return counter;
};

increment(); // 1
increment(); // 2（実行するたびに結果が変わる）
increment(); // 3
```

```typescript
// 冪等じゃない（リストへの追加）
const items: string[] = [];
const addItem = (item: string): string[] => {
  items.push(item);
  return items;
};

addItem('apple'); // ['apple']
addItem('apple'); // ['apple', 'apple']（実行するたびに要素が増える）
addItem('apple'); // ['apple', 'apple', 'apple']
```

### REST API での冪等性

REST API の設計でよく出てくる概念です：

| メソッド | 冪等性   | 説明                                     |
| -------- | -------- | ---------------------------------------- |
| GET      | 冪等     | 何度読み取っても同じ結果                 |
| PUT      | 冪等     | 同じ内容で何度更新しても結果は同じ       |
| DELETE   | 冪等     | 削除済みのリソースを削除しても結果は同じ |
| POST     | 非冪等   | 実行するたびに新しいリソースが作成される |
| PATCH    | 実装次第 | 部分更新の内容による                     |

例えば：

```typescript
// POST: 冪等じゃない
POST /users { name: "Keeth" }
// 1回目: ユーザーID 1 が作成される
POST /users { name: "Keeth" }
// 2回目: ユーザーID 2 が作成される（結果が違う）

// PUT: 冪等
PUT /users/1 { name: "Keeth" }
// 1回目: ユーザーID 1 の名前が "Keeth" になる
PUT /users/1 { name: "Keeth" }
// 2回目: ユーザーID 1 の名前が "Keeth" になる（結果は同じ）

// DELETE: 冪等
DELETE /users/1
// 1回目: ユーザーID 1 が削除される
DELETE /users/1
// 2回目: すでに存在しないので何も起きない（結果は同じ: ユーザーID 1 は存在しない）
```

## ３つの違いを整理

ここまでの内容を表にまとめてみます．

| 概念           | 着目点     | 条件                                       |
| -------------- | ---------- | ------------------------------------------ |
| **純関数**     | 関数の性質 | ① 同じ入力 → 同じ出力<br>② 副作用なし      |
| **参照透過性** | 式の性質   | 式を評価結果で置き換えても動作が変わらない |
| **冪等性**     | 操作の性質 | 何度実行しても同じ結果になる               |

### 具体例で比較

では，具体的なコード例で３つの概念を比較してみましょう．

#### 例 1: 絶対値を返す関数

```typescript
const abs = (x: number): number => Math.abs(x);
```

- **純関数**: ○（同じ入力で同じ出力，副作用なし）
- **参照透過**: ○（`abs(-5)` を `5` に置き換えても動作は変わらない）
- **冪等性**: ○（何度実行しても同じ結果）

#### 例 2: カウンターを増やす関数

```typescript
let counter = 0;
const increment = (): number => {
  counter++;
  return counter;
};
```

- **純関数**: ×（外部の状態に依存，副作用あり）
- **参照透過**: ×（`increment()` を結果で置き換えると動作が変わる）
- **冪等性**: ×（実行するたびに結果が変わる）

#### 例 3: データベースのレコードを更新する関数

```typescript
const setUserName = (userId: number, name: string): void => {
  db.users.update({ id: userId }, { name: name });
};
```

- **純関数**: ×（データベースという外部状態を変更する副作用あり）
- **参照透過**: ×（外部状態に依存するため）
- **冪等性**: ○（同じ引数で何度実行しても最終結果は同じ）

ここが面白いポイントで，**冪等だけど純関数ではない** というケースがあるんですね！

#### 例 4: 現在時刻を返す関数

```typescript
const getCurrentTime = (): Date => new Date();
```

- **純関数**: ×（同じ入力（引数なし）でも実行のたびに結果が変わる）
- **参照透過**: ×（式を結果で置き換えると動作が変わる）
- **冪等性**: ×（実行するたびに異なる時刻を返す）

#### 例 5: HTTP の DELETE リクエスト

```typescript
const deleteUser = (userId: number): void => {
  fetch(`/api/users/${userId}`, { method: 'DELETE' });
};
```

- **純関数**: ×（外部システムに副作用あり）
- **参照透過**: ×（外部状態に依存）
- **冪等性**: ○（何度削除しても，そのユーザーは存在しない状態になる）

## React での応用

React では「コンポーネントを純粋に保つ」という哲学があります．これは純関数の概念を React コンポーネントに適用したものです．

```tsx
// 純粋なコンポーネント（推奨）
type UserProps = {
  name: string;
  age: number;
};

const User = ({ name, age }: UserProps) => {
  return (
    <div>
      <p>Name: {name}</p>
      <p>Age: {age}</p>
    </div>
  );
};
```

```tsx
// 純粋じゃないコンポーネント（非推奨）
let renderCount = 0;

type CounterProps = {
  count: number;
};

const Counter = ({ count }: CounterProps) => {
  renderCount++; // レンダリング中に外部の変数を変更（副作用）
  return (
    <div>
      Count: {count}, Rendered: {renderCount} times
    </div>
  );
};
```

一方，`useEffect` では副作用を扱いますが，クリーンアップ関数を使って**冪等性を確保する**ことが重要です：

```tsx
type UserProfileProps = {
  userId: string;
};

const UserProfile = ({ userId }: UserProfileProps) => {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    let cancelled = false;

    fetch(`/api/users/${userId}`)
      .then((res) => res.json())
      .then((data: User) => {
        if (!cancelled) {
          setUser(data);
        }
      });

    // クリーンアップ関数で冪等性を確保
    return () => {
      cancelled = true;
    };
  }, [userId]);

  return <div>{user?.name}</div>;
};
```

## まとめ

３つの概念の違いを整理すると：

- **純関数**: プログラミングにおける理想的な関数の形．副作用がなく，予測可能で，テストしやすい
- **参照透過性**: プログラムの推論を簡単にする性質．コンパイラの最適化にも役立つ
- **冪等性**: システムの安全性と信頼性を高める性質．特に API 設計や分散システムで重要

それぞれ異なる概念ですが，どれも保守性の高いコードを書くために重要な考え方です．特に：

- **関数型プログラミング** では純関数と参照透過性が重要
- **API 設計** では冪等性が重要
- **React** では純関数の概念をコンポーネントに適用しつつ，副作用の処理では冪等性を意識する

個人的には，これらの概念を意識するようになってから，バグが減り，テストが書きやすくなったなと実感しています．

ではでは 👋

## 参考

- [Pure function - Wikipedia](https://en.wikipedia.org/wiki/Pure_function)
- [Referential transparency - Wikipedia](https://en.wikipedia.org/wiki/Referential_transparency)
- [Idempotence - Wikipedia](https://en.wikipedia.org/wiki/Idempotence)
- [Keeping Components Pure - React](https://ja.react.dev/learn/keeping-components-pure)
- [MDN Web Docs - HTTP request methods](https://developer.mozilla.org/ja/docs/Web/HTTP/Methods)
