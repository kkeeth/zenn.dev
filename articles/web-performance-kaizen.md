---
title: "🚧 WIP: web パフォーマンス改善 tip 集 🚧"
emoji: "⏫"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Performance"]
published: false
---

> In 1982 Walter J. Doherty and Ahrvind J. Thadani published, in the IBM Systems Journal, a research paper that set the requirement for computer response time to be 400 milliseconds, not 2,000 (2 seconds) which had been the previous standard. When a human being’s command was executed and returned an answer in under 400 milliseconds, it was deemed to exceed the Doherty threshold, and use of such applications were deemed to be “addicting” to users.

URL: https://lawsofux.com/en/doherty-threshold/

このように，コンピュータの応答時間の理想は 400ms という主張もあるそうです．

# CSS 周り

https://dev.to/mnathani/two-lines-of-css-that-boosts-7x-rendering-performance-4mjd

```css
{
  content-visibility: auto;
  contain-intrinsic-size: 1px 5000px;
}
```

たったこの２行で 7倍レンダリングパフォーマンスが上がるそうです…（マジかよ）

:::message
２つ目の `contain-intrinsic-size` ですが，記事内でも言及されている通り [Can I use](https://caniuse.com/?search=contain-intrinsic-size) を見ますと Chromium ブラウザでのみ利用できるものとなりますのでご注意．
:::
