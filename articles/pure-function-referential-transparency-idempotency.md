---
title: '純関数・参照透過性・冪等性の違いが分からなくなったのでまとめる'
emoji: '💡'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['関数型プログラミング', 'typescript', 'react', '純関数', '冪等性']
published: true
---

こんにちは．最近関数型プログラミングの概念をちゃんと理解したい欲が出てきました．[Keeth](https://x.com/kuwahara_jsri) こと桑原です．

以前 [「なぜ React 哲学を守らないといけないのか？」](https://zenn.dev/keeth/articles/why-learning-react-philosophy) という記事を書いたのですが，執筆中に純関数の定義に触れたとき，「これって冪等性のこと？いや，参照透過性？というか，そもそも自分はこの 2 つの概念の定義が曖昧かも…」という疑問がふっと浮かびました．良い機会なので，違いを整理してみます．コード例は今回も独断と偏見で TypeScript で書いています．

※もし間違っている記述があればバンバンコメントいただけると嬉しいです 🙇‍♂️

## TL;DR

- 純関数・参照透過性・冪等性は似ているようで違う概念
- 冪等性には**数学的な意味**（`f(f(x)) === f(x)`）と**操作としての意味**（何度実行しても状態が同じ）の 2 つの顔がある
- 「冪等だけど純関数ではない」例（DB 更新など）も，「純関数だけど冪等ではない」例（`x + 1` など）もある
- 関数型プログラミングでは純関数と参照透過性，API 設計では冪等性が重要
- React では純関数の概念をコンポーネントに，副作用の処理では冪等性を意識する

## 純関数（Pure Function）とは？

まずは**純関数**から．これは関数型プログラミングの基礎中の基礎ですね．

純関数は以下の 2 つの条件を満たす関数です：

1. **同じ入力**（引数）を与えられたとき，常に**同じ出力**（戻り値）を返す（決定論的である）
2. 関数の実行中に**外部の状態を変更**したり（副作用），**外部の状態に依存**したりしない

:::message
なお，ここで言う**副作用**とは，戻り値を返す以外の何かしらの形で観測可能な影響を与えること（グローバル変数や引数の書き換え，DB への書き込み，`console.log` などの I/O）を指します．一方，外部の状態を**読み取る**だけの操作は副作用とは呼ばれないことが多いですが，同じ入力でも読み取った値次第で出力が変わってしまうため，条件 1（決定論的である）に反します．書き込みは条件 2 違反，読み取りは条件 1 違反，どちらにしても純関数ではなくなる，と整理すると分かりやすいです．この「決定論的である」という性質は，後述の冪等性と区別するうえでも重要です．
:::

### 純関数の例

```typescript
const add = (a: number, b: number): number => a + b;

add(1, 2); // 3
add(1, 2); // 3（何度呼んでも同じ結果）
```

```typescript
const double = (arr: number[]): number[] => arr.map((x) => x * 2);

const nums = [1, 2, 3];
double(nums); // [2, 4, 6]
console.log(nums); // [1, 2, 3]（元の配列は変更されていない）
```

### 純関数じゃない例

```typescript
// 外部の状態に依存
let multiplier = 2;
const multiply = (x: number): number => x * multiplier;

multiply(5); // 10
multiplier = 3;
multiply(5); // 15（同じ入力なのに結果が違う）
```

```typescript
// 副作用がある
let counter = 0;
const increment = (): number => {
  counter++; // グローバル変数を変更
  return counter;
};

increment(); // 1
increment(); // 2（同じ入力（引数なし）なのに結果が違う）
```

```typescript
// 外部環境や実行時刻に依存

console.log(Date.now()); // 実行時刻に依存
console.log(Math.random()); // 毎回ランダムな値
```

## 参照透過性（Referential Transparency）とは？

**参照透過性**は，プログラミングにおける式の性質を表す概念です．

**「式をその評価結果で置き換えても，プログラムの動作が変わらない」** という性質を指します．

### 参照透過な例

```typescript
const add = (a: number, b: number): number => a + b;
const multiply = (x: number, y: number): number => x * y;

// ※以下の result は「同じ式の書き換え」を示すための別々の例です

// この式は
const result = multiply(add(2, 3), 4);

// こう置き換えても同じ結果になる
const result = multiply(5, 4); // add(2, 3) を 5 に置き換えた

// さらにこう置き換えても同じ（それはそう）
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

// この式は呼び出すたびに count が進むため
const a = incrementAndGet(); // 1（このとき式の「評価結果」は 1）

// 「式 incrementAndGet() を評価結果の 1 に置き換えられる」なら，こう書いても同じはず
const x = a + a; // 1 + 1 = 2

// しかし元の式を素直に評価すると…
const y = incrementAndGet() + incrementAndGet(); // 2 + 3 = 5（置き換えた結果と一致しない）
```

`incrementAndGet()` は呼び出すたびに異なる値を返すので，式を「ある 1 つの評価結果」で置き換えることができません．つまり参照透過ではありません．

### 純関数と参照透過性の関係

実は，**純関数の呼び出し式は，必ず参照透過になります**．逆に言えば，**呼び出し式が参照透過にならない関数は，純関数ではありません**．

ただし，概念としては：

- **純関数**: 関数の性質
- **参照透過性**: 式の性質

と，着目点が微妙に違うんですね 🤔

## 冪等性（Idempotency）とは？

冪等性は，**同じ操作を何度実行しても，1 回実行した時と同じ結果（状態）になる** という性質です．

ただし「操作を重ねる」の読み方によって，冪等性という言葉は 2 つの意味で使われます：

- **数学的な意味**: 入力と出力の型が同じ関数 `f: A → A` について，**出力をもう一度入力に食わせても**結果が変わらない．数式で書くと `f(f(x)) === f(x)`．例: 絶対値 `abs`
- **操作としての意味**: 状態を持つ系への操作について，**同じ操作をもう一度実行しても**，2 回目以降の操作が 1 回目の作った状態に何の影響も与えない．例: DB のレコード更新，HTTP の PUT / DELETE

一見別物に見えますが，操作の方も「系の状態を受け取って新しい状態を返す関数 `step: State → State`」と見なせば `step(step(state)) === step(state)` のことなので，根っこはどちらも同じ `f(f(x)) === f(x)` です．違うのは，値そのものを入出力と見るか，系の状態を入出力と見るか，だけです．

:::message
**冪等性 ≠ 決定論性** に注意してください．「同じ入力で常に同じ出力を返す（決定論的）」は純関数の条件①であって，冪等性とは別物です．冪等性は「**操作を重ねがけしても，1 回だけ実行したときと結果が同じ**」かどうかを見ます．次の `abs` の例も，単に `abs(-5)` を 3 回呼んで同じ値になること（＝決定論性）ではなく，**`abs(abs(x))` のように適用を重ねても結果が変わらないこと**（数学的な意味の冪等性）がポイントです．
:::

### 冪等な例

```typescript
// 冪等な関数
const abs = (x: number): number => Math.abs(x);

abs(-5); // 5
abs(abs(-5)); // abs(5) = 5（適用を重ねても結果は変わらない）
abs(abs(abs(-5))); // 5（何度適用しても同じ）
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
increment(); // 2（実行を重ねるたびに状態 counter が変化していく）
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
| GET      | 冪等     | 読み取りのみでサーバの状態を変えない     |
| PUT      | 冪等     | 同じ内容で何度更新しても結果は同じ       |
| DELETE   | 冪等     | 削除済みのリソースを削除しても結果は同じ |
| POST     | 非冪等   | 実行するたびに新しいリソースが作成される |
| PATCH    | 実装次第 | 部分更新の内容による                     |

なお POST も，リクエストに `Idempotency-Key` のような一意なキーを付与し，サーバ側で重複実行を弾く設計にすれば冪等にできます（決済 API などでよく使われる手法です）．

例えば：

```text
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

ちなみに 2 回目の DELETE は，実装によってはステータスコード 404 が返ってきます．「レスポンスが違うなら冪等じゃないのでは？」と思うかもしれませんが，冪等性はレスポンスの一致ではなく**サーバ側の状態**についての性質なので，これでも冪等です．

:::message
もう 1 つ補足すると，HTTP の冪等性が問うのは厳密には「**意図された効果**（intended effect）」が同じかどうかです（[RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.2.2)．旧 RFC 7231 §4.2.2 の定義を引き継いだものです）．サーバがリクエストごとにアクセスログを記録したり，リビジョン履歴を保存したりしても，それは意図された効果の外なので冪等性は損なわれません．ここまで「状態が同じ」と書いてきたのも，正確には「**操作が意図した対象の状態**が同じ」と読んでください．例えば `updateUserName` が内部で `updated_at` カラムを毎回更新していても，意図された効果（名前の更新）が同じである限り冪等とみなします．
:::

## 3 つの違いを整理

ここまでの内容を表にまとめてみます．

| 概念           | 着目点     | 定義                                                       |
| -------------- | ---------- | ---------------------------------------------------------- |
| **純関数**     | 関数の性質 | ① 同じ入力なら常に同じ出力<br>② 副作用なし                 |
| **参照透過性** | 式の性質   | 式を評価結果で置き換えてもプログラムの動作が変わらない     |
| **冪等性**     | 操作の性質 | 同じ操作を何度実行しても 1 回実行した時と同じ結果（状態）になる |

### 具体例で比較

では，具体的なコード例で 3 つの概念を比較してみましょう．

#### 例 1: 絶対値を返す関数

```typescript
const abs = (x: number): number => Math.abs(x);
```

- **純関数**: ○（同じ入力で同じ出力，副作用なし）
- **参照透過**: ○（`abs(-5)` を `5` に置き換えても動作は変わらない）
- **冪等性**: ○（数学的な意味で冪等．`abs(abs(-5))` も `abs(-5)` と同じ `5`．状態も変えないので操作としても冪等）

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
- **冪等性**: ×（実行を重ねるたびに `counter` が増え続け，状態が変わっていく）

#### 例 3: データベースのレコードを更新する関数

```typescript
// 実際には名前だけ変更するための関数を書くケースは少ないとは思う
const updateUserName = (userId: number, name: string): void => {
  db.users.update({ id: userId }, { name: name });
};
```

- **純関数**: ×（データベースという外部状態を変更する副作用あり）
- **参照透過**: ×（式を戻り値で置き換えると，DB 更新という副作用ごと消えてしまう）
- **冪等性**: ○（同じ引数で何度実行しても最終結果は同じ）

ここが面白いポイントで，**冪等だけど純関数ではない** というケースがあるんですね！

#### 例 4: 現在時刻を返す関数

```typescript
const getCurrentTime = (): Date => new Date();
```

- **純関数**: ×（同じ入力（引数なし）でも実行のたびに結果が変わる）
- **参照透過**: ×（式を結果で置き換えると動作が変わる）
- **冪等性**: ○（操作としての意味で冪等．外部の状態を一切変えないので，何度実行しても系の状態は 1 回実行時と同じ．GET が冪等なのと同じ理屈）

「実行するたびに違う時刻が返ってくるのに冪等なの？」と引っかかった方は，前述の **冪等性 ≠ 決定論性** を思い出してください．`getCurrentTime` は決定論的ではありませんが，状態には何の影響も与えないので操作としては冪等です．ここで観測しているのは戻り値ではなく**外部の状態への効果**です（2 回目の呼び出しは違う時刻を返すので，戻り値まで観測に含めれば「1 回実行した時と同じ」とは言えなくなります）．HTTP の冪等性が intended effect だけを見るのと同じ絞り方ですね．なお `getCurrentTime` は引数を取らない（出力を入力に食わせられない）ので，数学的な意味の冪等性はそもそも問えません．純関数の条件と冪等性の条件が別物であることがよく分かる例だと思います．

#### 例 5: HTTP の DELETE リクエスト

```typescript
const deleteUser = (userId: number): void => {
  fetch(`/api/users/${userId}`, { method: 'DELETE' });
};
```

- **純関数**: ×（外部システムに副作用あり）
- **参照透過**: ×（式を置き換えると DELETE リクエスト自体が飛ばなくなる）
- **冪等性**: ○（何度削除しても，そのユーザーは存在しない状態になる）

#### 例 6: 1 を足す関数

```typescript
const inc = (x: number): number => x + 1;
```

- **純関数**: ○（決定論的・副作用なし）
- **参照透過**: ○（`inc(2)` を `3` に置き換えても動作は変わらない）
- **冪等性**: ×（数学的な意味で非冪等．`inc(inc(2))` は `4` で，`inc(2)` の `3` と一致しない）

例 3 とは逆に，こちらは **純関数だけど（数学的な意味で）冪等ではない** ケースです．なお `inc` は状態を一切変えないので，**操作としての意味**なら例 4 と同じ理屈で冪等です．純関数の呼び出しは状態を変えない以上，操作としては必ず冪等になるので，「純関数だけど冪等ではない」という対比が成り立つのは数学的な意味の方です．例 3 と例 6 を見比べると，純関数と冪等性は**どちらか一方だけ成り立つこともある，独立した概念**だと分かりますね 🙌

#### 例 1〜6 のまとめ

| 例                          | 純関数 | 参照透過 | 冪等性 |
| --------------------------- | :----: | :------: | :----: |
| 例 1: `abs`                 |   ○    |    ○     |   ○    |
| 例 2: `increment`           |   ×    |    ×     |   ×    |
| 例 3: `updateUserName`（DB）|   ×    |    ×     |   ○    |
| 例 4: `getCurrentTime`      |   ×    |    ×     |   ○    |
| 例 5: `deleteUser`（HTTP）  |   ×    |    ×     |   ○    |
| 例 6: `inc`（x + 1）        |   ○    |    ○     |   ×    |

※ 冪等性の列は，例 1・例 6 は数学的な意味，例 2〜5 は操作としての意味での判定です（例 6 も操作としてなら冪等）．

例 3〜5（操作としての意味で冪等だけど純関数でない）と例 6（純関数だけど数学的な意味で冪等でない）を見比べると，純関数と冪等性がお互いを含意しない別物だと分かります．

なお「例 3〜5 と例 6 では冪等性の意味が違うので，またいで比べるのはずるくない？」と思った方のために補足すると，どちらか片方の意味に固定しても純関数との関係は示せます．**数学的な意味に固定した場合**，例えばログを出力しながら `Math.abs(x)` を返す関数は，純関数ではありませんが値としては `f(f(x)) === f(x)` を満たします（純関数でないけど冪等）．例 1 の `abs`（純関数で冪等），例 6 の `inc`（純関数だけど冪等でない），カウンタを増やしつつ `x + 1` を返す関数（純関数でなく冪等でもない）と合わせて 4 象限すべてに例が作れるので，純関数と冪等性は独立です．一方 **操作としての意味に固定した場合**，純関数は状態を変えないので「純関数 ⇒ 冪等」の片方向含意が成り立ち，独立ではなくなります（逆は成り立たないのは例 3〜5 の通りです）．

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

:::message
**用語の注意**: React 公式ドキュメントの [Components and Hooks must be pure](https://ja.react.dev/reference/rules/components-and-hooks-must-be-pure) では「コンポーネントや Hook は **idempotent（冪等）** であるべき」と書かれていますが，そこで言う idempotent は「**同じ入力なら常に同じ結果を返す**」，つまり本記事で言う**決定論性**の意味です．実際 React 公式は，`new Date()` や `Math.random()` は同じ入力でも結果が変わるためレンダー中では idempotent ではない，と説明しています．本記事の冪等性（数学的な意味・操作としての意味）とは別物なので，React のドキュメントを読むときは読み替えてください．
:::

一方，`useEffect` では副作用を扱いますが，**effect を何度実行しても安全（再実行に耐える）ように設計する**ことが重要です．これはまさに，effect を冪等に近づけるという話です．React の Strict Mode は開発時に effect をわざと 2 回実行し，この再実行への耐性が欠けていないかを検出します．次の例ではクリーンアップ関数で `cancelled` フラグを立て，古いレスポンスで `state` を上書きしてしまう競合状態（race condition）を防いでいます：

```tsx
type User = {
  id: number;
  name: string;
};

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

    // クリーンアップ関数で再実行に備える（古いレスポンスでの上書きを防ぐ）
    return () => {
      cancelled = true;
    };
  }, [userId]);

  return <div>{user?.name}</div>;
};
```

## まとめ

3 つの概念の違いを整理すると：

- **純関数**: 同じ入力なら常に同じ出力を返し，副作用を持たない**関数**．予測可能でテストしやすい
- **参照透過性**: 式を評価結果で置き換えてもプログラムの動作が変わらないという**式**の性質．コードの推論が楽になり，コンパイラの最適化にも役立つ
- **冪等性**: 同じ操作を何度実行しても 1 回実行した時と同じ結果（状態）になるという**操作**の性質（数学的な意味では `f(f(x)) === f(x)`）．ネットワーク障害でリクエストをリトライしたり，同じメッセージが重複配信（at-least-once）されたりしても状態が壊れないので，API 設計や分散システムで特に重要

それぞれ異なる概念ですが，どれも保守性の高いコードを書くために重要な考え方です．特に：

- **関数型プログラミング** では純関数と参照透過性が重要
- **API 設計** では冪等性が重要
- **React** では純関数の概念をコンポーネントに適用しつつ，副作用の処理では冪等性を意識する

これで今まであいまいだったものがかなりスッキリしました．

ではでは 👋

## 参考

- [Pure function - Wikipedia](https://en.wikipedia.org/wiki/Pure_function)
- [Referential transparency - Wikipedia](https://en.wikipedia.org/wiki/Referential_transparency)
- [Idempotence - Wikipedia](https://en.wikipedia.org/wiki/Idempotence)
- [Keeping Components Pure - React](https://ja.react.dev/learn/keeping-components-pure)
- [MDN Web Docs - HTTP request methods](https://developer.mozilla.org/ja/docs/Web/HTTP/Methods)
- [MDN Web Docs - 冪等](https://developer.mozilla.org/ja/docs/Glossary/Idempotent)
