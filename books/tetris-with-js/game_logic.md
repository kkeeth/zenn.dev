---
title: "ステップ3: ゲームロジックの実装"
---

# 3.1 ゲームの状態を管理

まずはゲームの状態を管理するための変数を追加します．

※ここで画面上はエラーになります．

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

const createMatrix = (w, h) => {
  const matrix = [];
  while (h--) {
    matrix.push(new Array(w).fill(0));
  }
  return matrix;
}

const arena = createMatrix(12, 20);

const player = {
  pos: {x: 0, y: 0},
  matrix: null,
  score: 0,
};
```

# 3.2 衝突判定の実装

ピースが他のピースや壁と衝突するかどうかを判定する関数を作成します．

```js
const collide = (arena, player) => {
  const [m, o] = [player.matrix, player.pos];
  for (let y = 0; y < m.length; ++y) {
    for (let x = 0; x < m[y].length; ++x) {
      if (m[y][x] !== 0 &&
        (arena[y + o.y] &&
        arena[y + o.y][x + o.x]) !== 0) {
        if (y + o.y <= 0) {
          return true; // 上部に到達
        }
        return true;
      }
    }
  }
  return false;
}
```

# 3.3 ピースのリセットとゲームオーバー

ピースをリセットする際のロジックとゲームオーバーの判定を実装します．

```js
const playerReset = () => {
  const pieces = 'TJLOSZI';
  player.matrix = createPiece(pieces[pieces.length * Math.random() | 0]);
  player.pos.y = 0;
  player.pos.x = (arena[0].length / 2 | 0) - (player.matrix[0].length / 2 | 0);
  if (collide(arena, player)) {
    gameOver();
  }
}

const gameOver = () => {
  cancelAnimationFrame(animationId);
  document.getElementById('restartButton').style.display = 'block';
}
```

# 3.4 ピースの移動

ピースを左右に移動するロジックを実装します．

```js
const playerMove = (dir) => {
  player.pos.x += dir;
  if (collide(arena, player)) {
    player.pos.x -= dir;
  }
}
```

# 3.5 ピースの回転

ピースを回転するロジックを実装します．

```js
const rotate = (matrix, dir) => {
  for (let y = 0; y < matrix.length; ++y) {
    for (let x = 0; x < y; ++x) {
      [matrix[x][y], matrix[y][x]] = [matrix[y][x], matrix[x][y]];
    }
  }

  if (dir > 0) {
    matrix.forEach(row => row.reverse());
  } else {
    matrix.reverse();
  }
}

const playerRotate = (dir) => {
  const pos = player.pos.x;
  let offset = 1;
  rotate(player.matrix, dir);
  while (collide(arena, player)) {
    player.pos.x += offset;
    offset = -(offset + (offset > 0 ? 1 : -1));
    if (offset > player.matrix[0].length) {
      rotate(player.matrix, -dir);
      player.pos.x = pos;
      return;
    }
  }
}
```
