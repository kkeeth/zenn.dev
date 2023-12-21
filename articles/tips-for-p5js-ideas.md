---
title: "クリエイティブコーディングで使えそうなノイズを集めてみた"
emoji: "🎨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Tips", "Collection", "p5js"]
published: false
---

# Collection of ideas in p5js

## Draw a striped circle

```js
const diameter = 100;

function setup() {
  createCanvas(400, 400);
  background(255);
  noLoop();
  noStroke();

  translate(width / 2, height / 2);

  fill(0);
  ellipse(0, 0, diameter);

  fill(255);
  for (let i = -diameter / 2; i < diameter / 2; i += 10) {
    i % 20 === 0 && rect(i, -diameter / 2, 10, diameter);
  }
}
```

`with WEBGL`

```js
function setup() {
  createCanvas(400, 400, WEBGL);
  background(255);
  noLoop();
  noStroke();

  let pg = createGraphics(100, 100);
  pg.background(255);
  pg.fill(0);
  pg.rectMode(CENTER);
  for (let i = 0; i < 100; i += 10) {
    i % 20 === 0 && pg.rect(i, 0, 10, height);
  }

  texture(pg);
  ellipse(0, 0, 100);
}
```
