---
title: "🚧 WIP: web パフォーマンス改善 tip 集 🚧"
emoji: "⏫"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Performance"]
published: false
---


# CSS 周り

@[Two lines of CSS that boosts 7x rendering performance!](https://dev.to/mnathani/two-lines-of-css-that-boosts-7x-rendering-performance-4mjd)

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
