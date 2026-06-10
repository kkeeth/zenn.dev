---
title: 'Riot.js 向けのリアクティブなライブラリ riot-composables を作ったまとめ'
emoji: '⚙️'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['riotjs', 'riot']
published: false
---

:::message
本記事は [Riot.js Advent Calendar 2025](https://qiita.com/advent-calendar/2025/riotjs) の第 18 日目(遅刻組) の記事になります！
:::

## TL;DR

- Riot.js にロジック再利用の仕組みがない
- Vue 3 Composition API 風の riot-composables を作った
- React Hooks じゃなく Composables を選んだ理由と経緯を解説
- First Mover Advantage！競合ゼロの完全オリジナル実装

## 作ったもの

タイトル通り，riot-composables - Riot.js 用の Composables ライブラリ

特徴：

🎣 Vue 3 Composition API 風の API
⚡ Proxy ベースの自動リアクティビティ（this.update()不要）
🪶 Riot.js の軽量さを維持
📦 TypeScript 完全対応
🔧 カスタム composable 作成が簡単

GitHub リポジトリ： [https://github.com/kkeeth/riot-composables](https://github.com/kkeeth/riot-composables)

## Riot.js に「ない」ものを作る ー riot-composables 誕生秘話

こんにちは．[Keeth](https://x.com/kuwahara_jsri) こと桑原です．

今日は、軽量 UI ライブラリ「[Riot.js（以下、riot）](https://riot.js.org/)」に、Vue 3 風の Composables 機能を実装したお話をします．

「え、また新しいライブラリ？React でよくない？」と思ったあなた。ちょっと待ってください。この記事は、ライブラリを作る過程で「設計って何？」「既存のものとどう違わせる？」みたいな、わりと普遍的な悩みと向き合った記録でもあります。

## Riot.js って何？（知らない人向け）

Riot.js は、React や Vue と同じくコンポーネントベースの UI ライブラリです。最大の特徴は**軽量さ**。

```
React: 44.5KB (gzip)
Vue:   45KB (gzip)
Riot:  6KB (gzip)  ← 軽い！
```

ただし、軽い代わりに機能は必要最小限。「ちょうどいい塩梅」を目指している感じのライブラリです。

```riot
<!-- Riot.jsの例 -->
<my-counter>
  <p>Count: {state.count}</p>
  <button onclick={increment}>+</button>

  <script>
    export default {
      state: { count: 0 },
      increment() {
        this.state.count++
        this.update()  // 手動で更新を呼ぶ
      }
    }
  </script>
</my-counter>
```

シンプルですよね。でも、シンプルゆえに困ることもあります。

## 「ない」ものリスト

Riot.js を使っていて、ふと気づきました。

```
✅ ある: コンポーネント
✅ ある: 状態管理（state）
✅ ある: ライフサイクル（onMounted等）

❌ ない: Hooks的なもの
❌ ない: Composition API的なもの
❌ ない: ロジックを再利用する方法（Mixinはv4で削除）
❌ ない: Next.js/Nuxtみたいなフレームワーク
```

特に困ったのが**ロジックの再利用**。React なら Custom Hooks、Vue 3 なら Composables があるのに、Riot.js には...ない！

## そうだ、作ろう（単純）

最初に考えたのは「uhooks というライブラリがあるらしい」という情報。でも、使ってみると...

> **僕**: uhooks というライブラリがある。これを使えば riotjs にも hooks がインストールできるのだが、ちょっと使い勝手が悪く（初期インストールの仕組み的に）、カスタムフックスも作りにくい

> **Claude**: あなたの直感は正しいです。**Riot.js に Hooks や関数コンポーネントは構造的に相性が悪い**です。

理由：

- Riot.js はインスタンスベース（オブジェクト型）
- React Hooks は関数再実行が前提
- この 2 つは根本的に違う

なるほど...無理やり Hooks を移植するのは筋が悪いと。

## じゃあどうする？

ここで重要な判断をしました。

> **Claude**: むしろ Riot.js の**軽量さとシンプルさを活かした独自の状態管理システム**を作る方が、フレームワークとしての個性が出ると思いますがいかがでしょう？

> **僕**: はい。これは自分もうすうす感じていたことです。そもそもの作りが react とは違うし、hooks の概念を入れるのであれば、じゃあ react で書くわ、と判断されがちです。

そうなんです。**React のコピーを作っても意味がない**。Riot.js らしさを保ちながら、モダンなパターンを取り入れる方法を探すべきだと。

## 戦略的な選択：3 つの路線

議論の中で、3 つのアプローチが見えてきました。

### 案 1: Mixin 的なアプローチ

Riot.js v3 には Mixin 機能がありました。それを復活させる？

```javascript
// v3時代のMixin
export default {
  mixins: [formMixin, validationMixin],
  // ...
};
```

**メリット**: Riot.js の歴史的な資産を活かせる、v3 ユーザーの移行パスになる

### 案 2: Hooks 風 API（独立ライブラリ）

React っぽい見た目だけど、中身は Riot.js 用に最適化。

```javascript
import { useCounter } from 'riot-composables';

export default {
  onBeforeMount() {
    const counter = useCounter(this, 0); // thisを渡す
    Object.assign(this, counter);
  },
};
```

**メリット**: React ユーザーに親しみやすい、段階的に採用できる

### 案 3: Composition API 的なアプローチ（Vue 3 風）

```javascript
export default {
  onBeforeMount() {
    const { count, increment } = useCounter(this, 0);
    this.count = count;
    this.increment = increment;
  },
};
```

**メリット**: Vue 3 の Composition API に近い、setup 関数 1 回実行の思想が合う

## 決断：Composables で行く

最終的に、**案 2 と案 3 のハイブリッド**を選びました。

理由：

1. **名前は"Composables"** → React との差別化、Vue 3 からインスピレーション
2. **見た目は`useXxx`** → 開発者に馴染みやすい
3. **実行モデルは Vue 3 風** → Riot.js との相性が良い
4. **明示的な`this`渡し** → Riot.js のシンプルさを維持

```javascript
// 最終的な形
import { useReactive, useEffect } from 'riot-composables';

export default {
  onBeforeMount() {
    const state = useReactive(this, { count: 0 });

    useEffect(
      this,
      () => {
        console.log('Count:', state.count);
      },
      () => [state.count],
    );

    this.state = state;
  },
};
```

**見た目は React、中身は Vue 3、精神は Riot.js**という感じ。

## 段階的アプローチ：両方を目指す

さらに重要な決断がありました。

> **僕**: 両方を目指す。どちらもあって困る世界ではない

当初は「ライブラリ vs フレームワーク」で迷っていたんですが、「両方作ればいいじゃん」という結論に。

```
Phase 1: riot-composables（軽量ライブラリ）
└─ コアcomposables機能のみ、npm単体で公開

Phase 2: エコシステムパッケージ
├─ @riot-framework/router
├─ @riot-framework/fetch
└─ @riot-framework/forms

Phase 3: riot-framework（統合フレームワーク）
└─ Next.js/Nuxt的なオールインワン体験
```

まずは**riot-composables をしっかり作る**。その上でエコシステムを育てる。段階的に進めることで、リスクを減らしつつ、最終的には「Riot.js 版 Next.js」も視野に入れる作戦です。

## 競合調査：誰も作ってない！

ここで重要な発見がありました。

> **僕**: ちなみに、すでに同じこと考えてラッパー作っている人やリポジトリはある？

> **Claude**: 調査の結果：
> ✅ 状態管理ライブラリは複数ある（riotx、riot-state 等）
> ✅ ルーターラッパーは複数ある
> ❌ **Composables/Hooks 風のライブラリは見つからなかった**

完全にオリジナルです。競合がいない。これは...

**First Mover Advantage のチャンス！**

Riot.js の「デファクトスタンダード」composables 実装になれる可能性があります。

## なぜ Composables という名前？

ここで少し脇道。なぜ"Composables"なのか？

| 名前        | 意味                 | 誰が使ってる？ |
| ----------- | -------------------- | -------------- |
| Hooks       | フック（引っかける） | React          |
| Composables | 組み立て可能         | Vue 3          |

**Composable = Compose（組み立てる）+ able（できる）**

React の"Hooks"は強烈なブランドイメージがあります。もし"riot-hooks"という名前にしたら、「React のパクリでしょ？」と思われかねない。

一方、"Composables"は：

- Riot.js の"Open Stack"哲学と合致
- 「組み立て可能」という本質を表現
- Vue 3 の成功事例から学べる

そして何より、**React と Vue の良いとこ取りができる**自由度があります。

## 次回予告

ここまでで、riot-composables の**コンセプト**と**戦略**が固まりました。

次回は、実際の**設計と実装**の話をします：

- React Hooks との根本的な違いは？
- Proxy ベースのリアクティビティって何？
- `riot.install()`を使った 3 層アーキテクチャ
- カスタム composable の作り方

お楽しみに！

---

## 今日のまとめ

- Riot.js には**ロジック再利用の仕組みがない**（問題提起）
- 無理やり Hooks を移植するのは**筋が悪い**（方針決定）
- **Riot.js らしさを保ちながら Composition**を実現する（戦略）
- 名前は**Composables**、見た目は`useXxx`（ブランディング）
- **段階的アプローチ**で両方（ライブラリとフレームワーク）を目指す
- 競合はいない、**First Mover Advantage**のチャンス

→ 次回：技術的な設計と実装の話へ続く

## リンク

- [riot-composables GitHub リポジトリ](https://github.com/YOUR_USERNAME/riot-composables)（準備中）
- [Riot.js 公式サイト](https://riot.js.org/)
- [Vue 3 Composition API](https://vuejs.org/guide/reusability/composables.html)
