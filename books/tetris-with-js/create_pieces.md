---
title: "ステップ2: テトリスのピースを作成"
---

# 2.1 ピースのデータ構造を定義

各テトリスのピースを表すデータ構造を作成します．自分独自のピースを作成することも可能ですが，今回はよくあるテトリス標準のピースの作り方をそのまま採用するため，ピースは二次元配列で作成します．各ピースの色を塗るxyの位置の数字（例：`J` の場合は `4`）ですが，ピースの色も配列で持つ予定のため，数字はピースごとに異なります．

:::message
数字が `1` から始まるのは，`0` は「塗らないこと」と同義のため，色の配列の 0 番目は `null` にする予定です．
:::

```js
const createPiece = (type) => {
  if (type === 'T') {
    return [
      [0, 0, 0],
      [1, 1, 1],
      [0, 1, 0],
    ];
  } else if (type === 'O') {
    return [
      [2, 2],
      [2, 2],
    ];
  } else if (type === 'L') {
    return [
      [0, 3, 0],
      [0, 3, 0],
      [0, 3, 3],
    ];
  } else if (type === 'J') {
    return [
      [0, 4, 0],
      [0, 4, 0],
      [4, 4, 0],
    ];
  } else if (type === 'I') {
    return [
      [0, 5, 0, 0],
      [0, 5, 0, 0],
      [0, 5, 0, 0],
      [0, 5, 0, 0],
    ];
  } else if (type === 'S') {
    return [
      [0, 6, 6],
      [6, 6, 0],
      [0, 0, 0],
    ];
  } else if (type === 'Z') {
    return [
      [7, 7, 0],
      [0, 7, 7],
      [0, 0, 0],
    ];
  }
}
```

# 2.2 ピースの色

続いて先ほど言及した，各ピースの色を定義します．ここは適宜お好きな色を指定してください．

```js
const colors = [
  null,
  '#FF0D72',
  '#0DC2FF',
  '#0DFF72',
  '#F538FF',
  '#FF8E0D',
  '#FFE138',
  '#3877FF',
];
```

# 2.3 ピースの描画

続いてピースをキャンバスに描画する関数を作成します．

```js
/**
 * @param {object} matrix - ピースの形状と配置を表す2次元配列
 * @param {object} offset - ピースの描画位置を指定するオブジェクト．xとyのプロパティを持つ
 */
const drawMatrix = (matrix, offset) => {
  matrix.forEach((row, y) => {
    row.forEach((value, x) => {
      if (value !== 0) {
        context.fillStyle = colors[value];
        context.fillRect(x + offset.x, y + offset.y, 1, 1);
      }
    });
  });
}
```

`drawMatrix` の中では引数に渡されたピースの形状と位置を表す `matrix` の中身の二次元配列について， まずは行ごとに反復処理を行います．`row` が現在の行，`y` がその行のインデックスになります．さらに，現在の行（`row`）の各セルに対して反復処理を行います．`value` が現在のセルの値，`x` がそのセルのインデックスになります．

`value` の値（現在のセルの値）が 0 の場合は空のセル，0 でない場合はピースの一部を意味しますので，0 ではない場合，`context.fillStyle` で色を指定し，`context.fillRect` で描画をします．

```js
context.fillRect(x + offset.x, y + offset.y, 1, 1);
```

上記の処理は，指定された位置に 1x1 の矩形を描画しています．`x + offset.x` と `y + offset.y` は，それぞれX座標とY座標にオフセット（指定された幅と高さ）を加えた値です．これにより、ピースの描画位置が調整されます．

では実際に画面にピースを描画してみましょう．今回は位置とどのピースにするかはベタ書き固定で表示します．`app.js` 末尾に以下を追記してください．

```js
drawMatrix(createPiece('T'), { x: 5, y: 5 });
```

このように表示されたら完成です！

![ピースの表示](https://storage.googleapis.com/zenn-user-upload/1bc5b2122bca-20240707.png)