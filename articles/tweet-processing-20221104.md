---
title: "私が感動した Processing 製の作品のソースコードを解析してみた"
emoji: "🎨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["p5js", "processing", "JavaScript"]
published: false
---

こんにちは．Twitter には `#つぶやきProcessing` という魅力的なタグがあり，毎日数々の美しい作品がこのタグを付けて投稿されています．これを眺めるだけでも一日中過ごせるくらいです（個人の感覚です）．

今日はその中でも特に度肝を抜かれ，かつ感動した作品が 2022/11/04 に投稿されておりましたので，個人の拙い能力で解析に挑戦してみました💁

# 作品

なにはともあれ今回対象の作品．

https://twitter.com/yuruyurau/status/1588062547315679232?s=20&t=cqrHXW-7fE6o_7XvncAFKQ

初めてみたときは思わず言葉を失いました．いや，これ twitter のツイートで表現できるものですので，かなり文字数も少ないんですよ！なのにこの表現ってどうなってんの！？と…

これはかなり学びになると思い解析を試みようと思い立った次第です．それにしても美しい…

# ソースコード

では上記ツイートのコードを改めて書きます．

```js:整形前
$=[]
draw=_=>{$[0]??createCanvas(540,540);background(0,9);$=$.map((v,i)=>stroke(i,i/3,i/5).point(v.copy().add(2,1.6).mult(135))+v.add(sin(v.y*(r=(v.x*2+2.5^v.y+2)*8))/90,cos(v.x*r)/90))[2e3]?$.slice(-1980):[...$,...[...Array(20)].map(p5.Vector.random3D)]}
```

まぁ，minify されているので見辛いですよね😅 ということで整形しました．

```js:整形後
$ = [];
draw = (_) => {
  $[0] ?? createCanvas(540, 540);
  background(0, 9);
  $ = $.map(
    (v, i) =>
      stroke(i, i / 3, i / 5).point(v.copy().add(2, 1.6).mult(135)) +
      v.add(
        sin(v.y * (r = ((v.x * 2 + 2.5) ^ (v.y + 2)) * 8)) / 90,
        cos(v.x * r) / 90,
      ),
  )[2e3]
    ? $.slice(-1980)
    : [...$, ...[...Array(20)].map(p5.Vector.random3D)];
};
```

もう少し分かりやすくするため愚直に書き出すと，こんな感じ💁

```js:愚直に書き出し
let $ = [];
const num = 2e3;

function setup() {
  createCanvas(600, 600);
}

function draw() {
  background(0, 9);

  const arr = $.map((v, i) => {
    stroke(i, i / 3, i / 5);
    point(v.copy().add(2, 1.6).mult(135));
    r = ((v.x * 2 + 2.5) ^ (v.y + 2)) * 8;

    return v.add(sin(v.y * r) / 90, cos(v.x * r) / 90);
  });

  if (arr[num]) {
    $ = $.slice(-(num - 20));
  } else {
    let newArr = [];

    for (let i = 0; i < 20; i++) {
      newArr.push(p5.Vector.random3D());
    }
    $ = [...$, ...newArr];
  }
}
```

本記事では基本的にこちらの愚直バージョンを用いて解説していきますー(*･ω･)ﾉ


# 解析ポイント

計算処理もそうですが，今回は

1. __ベクトルを使って表現されていること__
2. __円の表現の工夫__

の２点が重要だなと感じました．その辺りを意識して色々分析したものを書いていきます．

## ① `p5.Vector.random3D`

まずは大元となるベクトルのインスタンスの生成に使われているこちらのメソッド．単にベクトルのインスタンスを生成したいのであれば `createVector()` でも良かったのですが，どうも __円の表現__ に絡んでいるのはここしかなさそうだなと感じておりまして．

試しに以下のように途中処理を排除して何が描かれるか見てみました💁

```js:コードの一部抜粋
  const arr = $.map((v, i) => {
    stroke(i, i / 3, i / 5);
    point(v.copy().add(2, 1.6).mult(135));
    r = 1; //((v.x * 2 + 2.5) ^ (v.y + 2)) * 8;

    return v; //.add(sin(v.y * r) / 12, cos(v.x * r) / 12);
  });

  if (arr[num]) {
    $ = $.slice(-(num - 20));
  } else {
    let newArr = [];

    for (let i = 0; i < 20; i++) {
      newArr.push(p5.Vector.random3D());
    }
    $ = [...$, ...newArr];
  }
```

この場合のプロットはこんな感じ．

![ranndom3D のプロット](https://storage.googleapis.com/zenn-user-upload/3c5408aebf6d-20221107.png)



というわけで，実際に[メソッドの定義](https://github.com/processing/p5.js/blob/v1.5.0/src/math/p5.Vector.js#L2033) を見てみましょう．

```js:random3D
/**
 * Make a new random 3D unit vector.
 *
 * @method random3D
 * @static
 * @return {p5.Vector} A new <a href="#/p5.Vector">p5.Vector</a> object
 * @example
 * <div class="norender">
 * <code>
 * let v = p5.Vector.random3D();
 * // May make v's attributes something like:
 * // [0.61554617, -0.51195765, 0.599168] or
 * // [-0.4695841, -0.14366731, -0.8711202] or
 * // [0.6091097, -0.22805278, -0.7595902]
 * print(v);
 * </code>
 * </div>
 */
p5.Vector.random3D = function random3D() {
  const angle = Math.random() * constants.TWO_PI;
  const vz = Math.random() * 2 - 1;
  const vzBase = Math.sqrt(1 - vz * vz);
  const vx = vzBase * Math.cos(angle);
  const vy = vzBase * Math.sin(angle);
  return new p5.Vector(vx, vy, vz);
};
```

ん〜なるほど！ランダムで `TWO_PI(2π)` までの値を取得していること，またこちらの式 `Math.sqrt(1 - vz * vz)` なんてまさに __原点を中心とした半径１の円の方程式__[^2] にしか見えませんw

[^2]: 一般的な $x^2 + y^2 = 1$ というもので，これの `x` か `y` についての式に変形すると導出できます．

ちなみに，ここの定義を愚直に `createVector()` メソッドを用いて以下のように（引数はテキトー）書いてみましたが，

```js
createVector(random(), random(), random() * 2 - 1)
```

全く違うプロットになりましたので，`p5.Vector.random3D()` メソッドを使うことがポイントのようです．

![createVector でのプロット](https://storage.googleapis.com/zenn-user-upload/0587dd9d526e-20221106.png)


:::details 余談（`p5.Vector.random2D()` メソッド）
勘の良い方は，今回は結局 2D の絵を描いているのだから `p5.Vector.random2D()` というメソッドを用いれば良くない？と思われたかと思います．そのご指摘はまっこと仰るとおりで実際に私も試してみましたが，どうもこちらでも違ったプロットになってしまいました．

![](https://storage.googleapis.com/zenn-user-upload/c86ec459f0b5-20221106.png)

これはこれで美しいですけどね😆
一応こちらも [メソッドの定義](https://github.com/processing/p5.js/blob/v1.5.0/src/math/p5.Vector.js#L1980) を見てみました．

```js
/**
 * Make a new 2D unit vector from a random angle.
 *
 * @method random2D
 * @static
 * @return {p5.Vector} A new <a href="#/p5.Vector">p5.Vector</a> object
 * @example
 * <div class="norender">
 * <code>
 * let v = p5.Vector.random2D();
 * // May make v's attributes something like:
 * // [0.61554617, -0.51195765, 0.0] or
 * // [-0.4695841, -0.14366731, 0.0] or
 * // [0.6091097, -0.22805278, 0.0]
 * print(v);
 * </code>
 * </div>
 *
 * <div>
 * <code>
 * function setup() {
 *   frameRate(1);
 * }
 *
 * function draw() {
 *   background(240);
 *
 *   let v0 = createVector(50, 50);
 *   let v1 = p5.Vector.random2D();
 *   drawArrow(v0, v1.mult(50), 'black');
 * }
 *
 * // draw an arrow for a vector at a given base position
 * function drawArrow(base, vec, myColor) {
 *   push();
 *   stroke(myColor);
 *   strokeWeight(3);
 *   fill(myColor);
 *   translate(base.x, base.y);
 *   line(0, 0, vec.x, vec.y);
 *   rotate(vec.heading());
 *   let arrowSize = 7;
 *   translate(vec.mag() - arrowSize, 0);
 *   triangle(0, arrowSize / 2, 0, -arrowSize / 2, arrowSize, 0);
 *   pop();
 * }
 * </code>
 * </div>
 */
p5.Vector.random2D = function random2D() {
  return this.fromAngle(Math.random() * constants.TWO_PI);
};
```

Doc がなげー！のに処理はたった１行やんけ！しかし，やはりこちらもランダムに `TWO_PI` までの値を取得して利用していますねー．

ということで，依存している [`fromAngle()` メソッドの定義](https://github.com/processing/p5.js/blob/00821f33ca1d8a6990364568f0374c4aaf713faa/src/math/p5.Vector.js#L1924) も見てみましょう．こちらも Doc が長いですが（苦笑）

```js
/**
 * Make a new 2D vector from an angle.
 *
 * @method fromAngle
 * @static
 * @param {Number}     angle The desired angle, in radians (unaffected by <a href="#/p5/angleMode">angleMode</a>)
 * @param {Number}     [length] The length of the new vector (defaults to 1)
 * @return {p5.Vector}       The new <a href="#/p5.Vector">p5.Vector</a> object
 * @example
 * <div>
 * <code>
 * function draw() {
 *   background(200);
 *
 *   // Create a variable, proportional to the mouseX,
 *   // varying from 0-360, to represent an angle in degrees.
 *   let myDegrees = map(mouseX, 0, width, 0, 360);
 *
 *   // Display that variable in an onscreen text.
 *   // (Note the nfc() function to truncate additional decimal places,
 *   // and the "\xB0" character for the degree symbol.)
 *   let readout = 'angle = ' + nfc(myDegrees, 1) + '\xB0';
 *   noStroke();
 *   fill(0);
 *   text(readout, 5, 15);
 *
 *   // Create a p5.Vector using the fromAngle function,
 *   // and extract its x and y components.
 *   let v = p5.Vector.fromAngle(radians(myDegrees), 30);
 *   let vx = v.x;
 *   let vy = v.y;
 *
 *   push();
 *   translate(width / 2, height / 2);
 *   noFill();
 *   stroke(150);
 *   line(0, 0, 30, 0);
 *   stroke(0);
 *   line(0, 0, vx, vy);
 *   pop();
 * }
 * </code>
 * </div>
 */
p5.Vector.fromAngle = function fromAngle(angle, length) {
  if (typeof length === 'undefined') {
    length = 1;
  }
  return new p5.Vector(length * Math.cos(angle), length * Math.sin(angle), 0);
};
```

こちらは `random3D()` メソッドよりも直接的で，`sin, cos` の値を引数にベクトルのインスタンスを生成していました．

どちらも共通しているのは初期生成するベクトルの座標は円の表現をするためのものでした．

:::

## ② `^` （排他的論理和）

お次は普段見ることが殆どない（個人の感覚です）+ __私は意図的に使ったことがない（のでどういう時に利用するのか，再現性がない）__ 演算子である，排他的論理和を意味する `^` です．電気回路に触れたことがある方は `XOR` と言った方がわかりやすいかもしれません．

排他的論理和そのものは説明できます（が今回は割愛します）が，先程申し上げたとおり利用したことがないため，これを用いた理由がわからず…こちらはどなたかわかる方ご教示いただけますと大変に助かります 🙇

ちなみにここの値を色々変えてみるとそれはそれで面白かったので試してみてください．固定値（今回は `8`）の大小によって四角になるものの数が増減します．

以下は `r = ((v.x * 2 + 2.5) ^ (v.y + 2)) * 20` にしたときの例．

![](https://storage.googleapis.com/zenn-user-upload/42b099337b17-20221107.png)

さらに変数を全て取っ払って固定で `r=20` にしたときの例．

![](https://storage.googleapis.com/zenn-user-upload/abc04c98b09c-20221107.png)

## ③ 変化量を微小に

対象のコードはこちら．

```js
// r は上記③で計算した値
return v.add(sin(v.y * r) / 90, cos(v.x * r) / 90);
```

ここの `/ 90` の部分がそれで，ここを削除するとわかりますが流れていくようなプロットにはなりません．これは元々の座標位置の値がランダムですので，差分を小さくしないと次の描画位置も見た目上はバラバラになってしまいます．ですので，小さくするために `/ 90` で小さくしていると思われます．値を 0 に近づけると挙動が分かりやすいです．[^1]

[^1]: 個人的には `/ 10 - 12` 辺りがいい感じで崩れていて美しいと感じています😆

## ④ 三角関数 + r（という名前の）変数 + xy

これらが揃っていると真っ先に想起されるものは __極座標__ です．`cos` の引数に `x` が，`sin` の引数に `y` が使われている辺りがそれだろうと（笑）しかし，今回は `r` が三角関数の引数として使われているので違うのかなと．

## ⑤ 小技

`stroke()` メソッドのレスポンスについて全く調べたことなかったのですが，よく考えれば `p5js` の組み込みメソッドなんだから `prototype` で他のメソッドもコールできておかしくないですよね．ということで，メソッドチェーンで `stroke().point()` とコールすることもできるんですね．

さらに，ベクトルインスタンスも同様で，インスタンスメソッドを `v.copy().add(2, 1.6).mult(135)` のようにメソッドチェーンで複数個つなげて処理することも可能なんですね．勉強になりました．

ちなみに今回は `point()` で点描画する際にコピーを生成し，そのコピーの値を

* 加算（`.add(2, 1.6）`: 画面上での描画位置を微調整
* 乗算（`.mult(135)`: インスタンスの座標位置が画面外にいるので無理やり画面内にずらす

と，されていました．

＃ 終わりに