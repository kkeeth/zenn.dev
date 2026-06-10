---
title: "様々なノイズを集めた"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["p5js", "noise"]
published: false
---

こんにちは．駆け出しジェネラティブアーティストの [くらふなアート](https://twitter.com/ArtmanKKeeth)[^1] こと桑原です．

この世界に飛び込む前はランダムなものって [random()](https://p5js-i18n-ja.pages.dev/ja/reference/#/p5/random) 関数しか知らなかったのですが，いざ飛び込んでみたところ [パーリンノイズ](https://ja.wikipedia.org/wiki/%E3%83%91%E3%83%BC%E3%83%AA%E3%83%B3%E3%83%8E%E3%82%A4%E3%82%BA) という技法を表した関数 [noise()](https://p5js-i18n-ja.pages.dev/ja/reference/#/p5/noise) というものを知りまして，とても感動したのを覚えています．

でしばらく `random(), noise()` で遊んでいたのですが，どうやらノイズは他にもいくつか存在するらしいぞ？ということを知りまして，１つ１つちゃんと把握したくなったのと，まとめた記事が欲しくなったので今書いている次第です．

ではでは行きましょう🙋

# パーリンノイズ（Perlin Noise）

冒頭で述べたノイズですね．p5.js でも `noise()` として実装されておりとても簡単に利用できてありがたいお話です．

1983 年に Ken Perlin 氏が映画『トロン』の CG 制作のために考案したノイズで，その功績により 1997 年にアカデミー科学技術賞を受賞しています．`random()` が完全にバラバラな値を返すのに対し，パーリンノイズは **隣り合う値がなめらかにつながる（連続性のある）** のが最大の特徴です．この性質のおかげで，地形・雲・炎・水面といった自然物の表現にうってつけなんですね．

p5.js では `noise()` に 1〜3 個の引数を渡すことで，1〜3 次元のノイズを取得できます．返り値は `0.0 〜 1.0` の範囲です．

```javascript
function setup() {
  createCanvas(600, 400);
}

function draw() {
  background(255);
  noStroke();
  fill(0);

  let xoff = 0;
  // x 座標を少しずつずらしながらノイズ値を取得し，なめらかな曲線を描く
  for (let x = 0; x < width; x++) {
    // frameCount を足すことで，時間方向にも変化させる
    const n = noise(xoff + frameCount * 0.01);
    const y = n * height;
    ellipse(x, y, 4, 4);
    // この刻み幅（増分）が小さいほどなめらかに，大きいほど荒くなる
    xoff += 0.01;
  }
}
```

`xoff` の増分を `0.01` から `0.1` などに変えると，同じパーリンノイズでもガクガクと荒れた見た目になります．この「サンプリングする間隔」がノイズの印象を大きく左右するので，ぜひ手元で調整して遊んでみてください．

# カールノイズ（Curl Noise）

[こちらの記事](https://webgl.souhonzan.org/entry/?v=0247) に冒頭一言でズバッと書いてあった以下がわかりやすかったので引用いたします．

> いわゆる一般的なパーリンノイズなどをベースに、ノイズの力で流体のような表現を行うことができるカールノイズ。

他にもいくつかの記事を参照しましたが，だいたい流体表現のために利用されている人が多かったです．実態としてはベクトルが計算できるノイズ関数だそうで，[こちらの記事](https://qiita.com/nyamadandan/items/2a8bc7a3639e7b5ce9c9) が参考になりますので，解説はこちらをご参照ください．中身見ると `ベクトル場` とか，`行列計算` とかが出てきてちょっとテンションは上がりますw

流体表現に使われる，ということからお察しのとおり，`WebGL`・`GLSL` 文脈で利用されるものですね．

簡単に仕組みを説明すると，あるノイズ場（スカラー場）から **回転（curl, ローテーション）を計算してベクトルを取り出す** のがカールノイズです．curl 演算によって得られるベクトル場は「発散がゼロ（湧き出しも吸い込みもない）」という性質を持つため，まるで非圧縮性の流体のような，渦を巻きながらも破綻しない自然な流れが表現できます．

p5.js（2D）で簡易的に再現するなら，パーリンノイズの偏微分（隣接点との差分）を取って，それを 90 度回転させてあげればカールノイズっぽいベクトル場が作れます．以下はパーティクルをそのベクトル場に沿って流すサンプルです．

```javascript
const particles = [];
const EPS = 0.0001; // 偏微分を取るための微小量
const SCALE = 0.005; // ノイズ空間のスケール

function setup() {
  createCanvas(600, 600);
  background(0);
  for (let i = 0; i < 1000; i++) {
    particles.push(createVector(random(width), random(height)));
  }
}

// 座標 (x, y) におけるカールノイズのベクトルを返す
function curl(x, y) {
  // x 方向・y 方向の偏微分を中心差分で近似
  const n1 = noise((x + EPS) * SCALE, y * SCALE);
  const n2 = noise((x - EPS) * SCALE, y * SCALE);
  const n3 = noise(x * SCALE, (y + EPS) * SCALE);
  const n4 = noise(x * SCALE, (y - EPS) * SCALE);

  const dx = (n1 - n2) / (2 * EPS);
  const dy = (n3 - n4) / (2 * EPS);

  // 勾配ベクトル (dx, dy) を 90 度回転させたものが curl
  return createVector(dy, -dx);
}

function draw() {
  noStroke();
  fill(255, 20);

  for (const p of particles) {
    const v = curl(p.x, p.y);
    p.add(v.mult(2));

    // 画面外に出たらランダムな位置へ戻す
    if (p.x < 0 || p.x > width || p.y < 0 || p.y > height) {
      p.set(random(width), random(height));
    }
    ellipse(p.x, p.y, 1.5, 1.5);
  }
}
```

実行すると，パーティクルが渦を巻きながら流れていく様子が観察できます．本格的に GPU で大量のパーティクルを動かしたい場合は，このロジックを `GLSL` のシェーダーに移植する形になります．

# シンプレクスノイズ（Simplex Noise）

これまた Ken Perlin 氏が，2001 年に自身の作ったパーリンノイズの欠点を改良するために発表したノイズです．本人による改良版というのが胸アツですね．

パーリンノイズの弱点として，

- 次元が増えると計算量が爆発的に増える（N 次元で $2^N$ 個の格子点を参照する必要がある）
- 格子状の構造に起因する方向性のアーティファクト（人工的な模様）が見えてしまう

といった点がありました．シンプレクスノイズは格子（正方形・立方体）の代わりに **シンプレクス（単体：2D なら正三角形，3D なら正四面体）** で空間を分割することで，N 次元でも参照点を $N + 1$ 個に抑え，計算量を大幅に削減しています．見た目も等方的（どの方向から見ても自然）になります．

ただし p5.js には標準では搭載されていません💦 そのため [simplex-noise](https://www.npmjs.com/package/simplex-noise) のようなライブラリを併用するのが手軽です．

```javascript
// CDN 例: https://cdn.jsdelivr.net/npm/simplex-noise@4.0.0/dist/esm/simplex-noise.min.js
import { createNoise2D } from 'simplex-noise';

const noise2D = createNoise2D();

function setup() {
  createCanvas(400, 400);
  pixelDensity(1);
  noLoop();
}

function draw() {
  loadPixels();
  const scale = 0.02;
  for (let x = 0; x < width; x++) {
    for (let y = 0; y < height; y++) {
      // simplex-noise の返り値は -1.0 〜 1.0 なので 0 〜 255 に変換する
      const n = noise2D(x * scale, y * scale);
      const bright = map(n, -1, 1, 0, 255);
      const idx = (x + y * width) * 4;
      pixels[idx] = bright;
      pixels[idx + 1] = bright;
      pixels[idx + 2] = bright;
      pixels[idx + 3] = 255;
    }
  }
  updatePixels();
}
```

# オープンシンプレクスノイズ（OpenSimplex Noise）

上のシンプレクスノイズ，実は **3D 以上のバージョンに特許** がかけられていました（2022 年に失効済み）．OSS でゲームを作っている人などにとってはこれが地味に厄介で，「特許に触れずにシンプレクスノイズ相当の品質が欲しい！」というニーズがありました．

そこで Kurt Spencer 氏が 2014 年に，特許を回避しつつシンプレクスノイズの欠点も改善する目的で発表したのが **OpenSimplex Noise** です．

シンプレクスノイズが空間を単体（シンプレクス）で分割するのに対し，OpenSimplex は格子をうまく傾けた **より対称性の高い分割** を用いることで，

- 特許を回避できる
- 方向性のアーティファクト（縞模様っぽいムラ）がさらに目立ちにくい

といった特徴を持ちます．後継として高速化・高品質化を図った **OpenSimplex2** も登場しており，現在新規で採用するならこちらが有力です．

p5.js では，これまた標準にはないのでライブラリを使います．[open-simplex-noise](https://www.npmjs.com/package/open-simplex-noise) などが使いやすいです．

```javascript
// CDN 例: https://cdn.jsdelivr.net/npm/open-simplex-noise@2.5.0/dist/open-simplex-noise.min.js
// グローバルに openSimplexNoise が生える

let noise2D;

function setup() {
  createCanvas(400, 400);
  pixelDensity(1);
  noLoop();
  // 引数はシード値．同じシードなら毎回同じノイズが得られる
  const gen = openSimplexNoise(12345);
  noise2D = gen.noise2D;
}

function draw() {
  loadPixels();
  const scale = 0.02;
  for (let x = 0; x < width; x++) {
    for (let y = 0; y < height; y++) {
      // 返り値は -1.0 〜 1.0
      const n = noise2D(x * scale, y * scale);
      const bright = map(n, -1, 1, 0, 255);
      const idx = (x + y * width) * 4;
      pixels[idx] = bright;
      pixels[idx + 1] = bright;
      pixels[idx + 2] = bright;
      pixels[idx + 3] = 255;
    }
  }
  updatePixels();
}
```

シンプレクスノイズのサンプルとほぼ同じインターフェースで使えるので，差し替えもラクですね．見た目の違いは正直パッと見では分かりにくいですが，「特許フリーで安心して使える」というのが実用上は何より嬉しいポイントです．

# バリューノイズ（Value Noise）

ここからは少し毛色が変わって，パーリンノイズの「親戚」のような存在を 2 つ紹介します．まずはバリューノイズから．

バリューノイズは，**格子点（グリッドの交点）にランダムな「値（value）」そのものを割り当てて，その間を補間する** という，とてもシンプルな仕組みのノイズです．

1. 格子点ごとに `random()` で 0〜1 の値を決める
2. 格子の内側の点は，周囲の格子点の値を **補間（線形補間やスムーズ補間）** して求める

実装が直感的で軽量なのが利点ですが，補間の都合上どうしても格子のブロック感が残りやすく，パーリンノイズに比べるとややのっぺりした印象になります．以下は 2D バリューノイズを p5.js で自前実装した例です．

```javascript
const GRID = 8; // 格子の数
const lattice = []; // 格子点ごとのランダム値

function setup() {
  createCanvas(400, 400);
  pixelDensity(1);
  noLoop();

  // 格子点に乱数を割り当てる
  for (let i = 0; i <= GRID; i++) {
    lattice[i] = [];
    for (let j = 0; j <= GRID; j++) {
      lattice[i][j] = random();
    }
  }
}

// スムーズな補間関数（スムーズステップ）
function fade(t) {
  return t * t * (3 - 2 * t);
}

function valueNoise(x, y) {
  const gx = x * GRID;
  const gy = y * GRID;
  const x0 = floor(gx);
  const y0 = floor(gy);
  const fx = fade(gx - x0);
  const fy = fade(gy - y0);

  // 周囲 4 つの格子点の値を 2 段階で補間する（バイリニア補間）
  const top = lerp(lattice[x0][y0], lattice[x0 + 1][y0], fx);
  const bottom = lerp(lattice[x0][y0 + 1], lattice[x0 + 1][y0 + 1], fx);
  return lerp(top, bottom, fy);
}

function draw() {
  loadPixels();
  for (let x = 0; x < width; x++) {
    for (let y = 0; y < height; y++) {
      const n = valueNoise(x / width, y / height);
      const idx = (x + y * width) * 4;
      const bright = n * 255;
      pixels[idx] = bright;
      pixels[idx + 1] = bright;
      pixels[idx + 2] = bright;
      pixels[idx + 3] = 255;
    }
  }
  updatePixels();
}
```

# グラデーションノイズ（Gradient Noise）

そして実は，**パーリンノイズはこのグラデーションノイズの一種** です．バリューノイズと対比させると違いがわかりやすいので，ここで整理しておきましょう．

- **バリューノイズ**：格子点に「値（スカラー）」を割り当てて補間する
- **グラデーションノイズ**：格子点に「勾配（ベクトル＝向き）」を割り当て，そのベクトルと格子点から対象点へのベクトルとの **内積** を取って補間する

つまり，格子点に置くものが「値」なのか「傾き（グラデーション）」なのか，というのが両者の本質的な違いです．グラデーションノイズは値ではなく傾きを補間するため，バリューノイズで気になりやすいブロック感が抑えられ，より自然でなめらかな見た目になります．パーリンノイズが自然物表現で重宝されるのは，まさにこの仕組みのおかげなんですね．

```javascript
const GRID = 8;
const grads = []; // 格子点ごとの勾配ベクトル（単位ベクトル）

function setup() {
  createCanvas(400, 400);
  pixelDensity(1);
  noLoop();

  // 各格子点にランダムな向きの単位ベクトルを割り当てる
  for (let i = 0; i <= GRID; i++) {
    grads[i] = [];
    for (let j = 0; j <= GRID; j++) {
      const angle = random(TWO_PI);
      grads[i][j] = createVector(cos(angle), sin(angle));
    }
  }
}

function fade(t) {
  return t * t * t * (t * (t * 6 - 15) + 10); // パーリン氏の 5 次の fade 関数
}

// 格子点 (ix, iy) の勾配と，対象点への距離ベクトルとの内積
function dotGrid(ix, iy, x, y) {
  const d = createVector(x - ix, y - iy);
  return grads[ix][iy].dot(d);
}

function gradientNoise(x, y) {
  const gx = x * GRID;
  const gy = y * GRID;
  const x0 = floor(gx);
  const y0 = floor(gy);
  const fx = fade(gx - x0);
  const fy = fade(gy - y0);

  // 4 隅の内積を補間する
  const top = lerp(dotGrid(x0, y0, gx, gy), dotGrid(x0 + 1, y0, gx, gy), fx);
  const bottom = lerp(dotGrid(x0, y0 + 1, gx, gy), dotGrid(x0 + 1, y0 + 1, gx, gy), fx);
  const n = lerp(top, bottom, fy); // -1 〜 1 付近
  return map(n, -1, 1, 0, 1);
}

function draw() {
  loadPixels();
  for (let x = 0; x < width; x++) {
    for (let y = 0; y < height; y++) {
      const n = gradientNoise(x / width, y / height);
      const idx = (x + y * width) * 4;
      const bright = n * 255;
      pixels[idx] = bright;
      pixels[idx + 1] = bright;
      pixels[idx + 2] = bright;
      pixels[idx + 3] = 255;
    }
  }
  updatePixels();
}
```

バリューノイズのサンプルと並べて実行すると，グラデーションノイズのほうがブロック感が少なくなめらかなことが見て取れると思います．ぜひ見比べてみてください👀

# その他の気になるノイズたち

ここまでが「格子ベースのノイズ」の主役級メンバーでした．ですがノイズの世界はまだまだ奥が深く，仕組みも見た目もガラッと変わる面白いものがたくさんあります．いくつかピックアップして紹介します🙋

## ウォーリーノイズ／セルラーノイズ（Worley / Cellular Noise）

個人的にイチオシのノイズです．1996 年に Steven Worley 氏が発表したもので，**空間にバラまいた特徴点（feature point）への距離** を使ってノイズを作ります．**ボロノイ図** や細胞（cell）のような模様が得られるのが特徴で，水面のコースティクス（光の網目模様），ひび割れ，石畳，爬虫類のウロコなどなど，自然界のあの「区画っぽい」模様を作るのに大活躍します．

仕組みはシンプルで，

1. 空間にランダムな点をいくつか配置する
2. 各ピクセルから「いちばん近い点までの距離」を求める
3. その距離を明るさにマッピングする

これだけです．「2 番目に近い点との距離の差」を使うとセルの境界線が浮き出てくるなど，バリエーションも豊富です．

```javascript
const points = [];
const N = 25; // 特徴点の数

function setup() {
  createCanvas(400, 400);
  pixelDensity(1);
  noLoop();
  for (let i = 0; i < N; i++) {
    points.push(createVector(random(width), random(height)));
  }
}

function draw() {
  loadPixels();
  for (let x = 0; x < width; x++) {
    for (let y = 0; y < height; y++) {
      // 最も近い特徴点までの距離を求める（F1 距離）
      let minDist = Infinity;
      for (const p of points) {
        const d = dist(x, y, p.x, p.y);
        if (d < minDist) minDist = d;
      }
      // 距離が近いほど暗く（細胞の中心が暗い）見える
      const bright = map(minDist, 0, 80, 0, 255, true);
      const idx = (x + y * width) * 4;
      pixels[idx] = bright;
      pixels[idx + 1] = bright;
      pixels[idx + 2] = bright;
      pixels[idx + 3] = 255;
    }
  }
  updatePixels();
}
```

上の素朴な実装は全ピクセル × 全点を総当たりするので重め（GLSL では格子分割で高速化するのが定番）ですが，模様の美しさはピカイチなのでぜひ一度試してほしいです．

## フラクタルブラウン運動（fBm / Fractal Brownian Motion）

厳密には「ノイズそのもの」というより **ノイズの使いこなしテクニック** なのですが，超頻出かつ強力なので紹介します．

やることは一言で言うと，**周波数（細かさ）と振幅（強さ）を変えたノイズを何枚も重ねる** だけです．大きくゆったりした波の上に，細かい波，さらに細かい波……と層（オクターブ）を積み重ねることで，一気に「自然物っぽいディテール」が生まれます．山の稜線や雲，地形生成でおなじみのアレですね．

```javascript
function fbm(x, y) {
  let value = 0;
  let amplitude = 0.5; // 振幅（重みづけ）
  let frequency = 1;   // 周波数（細かさ）
  const OCTAVES = 5;   // 重ねる枚数

  for (let i = 0; i < OCTAVES; i++) {
    value += amplitude * noise(x * frequency, y * frequency);
    frequency *= 2;    // 周波数は倍々に（細かく）
    amplitude *= 0.5;  // 振幅は半分ずつに（弱く）
  }
  return value;
}

function setup() {
  createCanvas(400, 400);
  pixelDensity(1);
  noLoop();
}

function draw() {
  loadPixels();
  const scale = 0.005;
  for (let x = 0; x < width; x++) {
    for (let y = 0; y < height; y++) {
      const n = fbm(x * scale, y * scale);
      const idx = (x + y * width) * 4;
      const bright = n * 255;
      pixels[idx] = bright;
      pixels[idx + 1] = bright;
      pixels[idx + 2] = bright;
      pixels[idx + 3] = 255;
    }
  }
  updatePixels();
}
```

`OCTAVES` を増やすほどディテールがリッチになります．ベースのノイズはパーリンでもシンプレクスでも何でも OK で，ここまで紹介してきたノイズと組み合わせて使えるのが嬉しいところです．

## ブルーノイズ（Blue Noise）

少し毛色の違う，**「点の配置（分布）」に関するノイズ** です．`random()` で点を打つとどうしても粗密のムラ（点が固まる場所とスカスカな場所）ができてしまいますが，ブルーノイズは **「どの点も互いに近づきすぎない，かつ均一に散らばっている」** という性質を持つ分布です．

名前の由来は，周波数解析したときに高周波（青い光の周波数帯）成分が強いことから．点描・スティップリング，ディザリング，サンプリング（レンダリングのアンチエイリアス）など，「ランダムだけど偏ってほしくない」場面で重宝されます．代表的な生成手法に **Poisson Disk Sampling** があります．

```javascript
// ごく素朴な「却下サンプリング」版の Poisson Disk もどき
// 既存の点から一定距離より近い候補は捨てる，を繰り返す
const points = [];
const R = 20;            // 点どうしの最小距離
const MAX_TRY = 3000;    // 試行回数

function setup() {
  createCanvas(400, 400);
  background(0);

  let tries = 0;
  while (tries < MAX_TRY) {
    const cand = createVector(random(width), random(height));
    let ok = true;
    for (const p of points) {
      if (dist(cand.x, cand.y, p.x, p.y) < R) {
        ok = false;
        break;
      }
    }
    if (ok) points.push(cand);
    tries++;
  }

  noStroke();
  fill(255);
  for (const p of points) ellipse(p.x, p.y, 4, 4);
}
```

`random()` でそのまま 200 点打ったものと見比べると，ブルーノイズのほうが「自然なのにムラがない」絶妙な散らばり方をしているのが分かります．本格的にやるなら Bridson のアルゴリズムを使うと高速です．

# 余談

> ノイズのアルゴリズムは本来、自然な je ne sais quoi （訳注：言葉では表せない質感、美しさ）をデジタルなテクスチャーに与えるために考え出されました。
>
> https://thebookofshaders.com/11/?lan=jp より

とあるように，自然界の美をどうやれば扱えるようになるのか，そのためのツールがノイズと言われると，個人的にはとてもしっくり来ました．

[^1]: アーティスト名も𝕏のアカウント名も迷走中です！w
