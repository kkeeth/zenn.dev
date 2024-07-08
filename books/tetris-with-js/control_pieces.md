---
title: "ステップ4: ゲームの更新と描画"
---

# 4.1 ピースの移動

ピースを左右に移動するロジックを実装します．

```js
const playerMove = (dir) => {
  player.pos.x += dir;
  if (collide(arena, player)) {
    player.pos.x -= dir;
  }
}
```

# 4.2 ピースの回転

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


# 4.2 ピースの落下

ピースを自然に落下させるロジックを実装します．

```js
const playerDrop = () => {
  player.pos.y++;
  if (collide(arena, player)) {
    player.pos.y--;
    merge(arena, player);
    playerReset();
    arenaSweep();
    updateScore();
  }
  dropCounter = 0;
}
```

# 4.3 ピースのマージ

ピースを固定するロジックを実装します．

```js
const merge = (arena, player) => {
  player.matrix.forEach((row, y) => {
    row.forEach((value, x) => {
      if (value !== 0) {
        arena[y + player.pos.y][x + player.pos.x] = value;
      }
    });
  });
}
```

# 4.4 ラインの消去

完成したラインを消去するロジックを実装します．

```js
const arenaSweep = () => {
  outer: for (let y = arena.length - 1; y > 0; --y) {
    for (let x = 0; x < arena[y].length; ++x) {
      if (arena[y][x] === 0) {
        continue outer;
      }
    }

    const row = arena.splice(y, 1)[0].fill(0);
    arena.unshift(row);
    ++y;

    player.score += 10;
  }
}
```

ここまででロジックは一通り実装し終わりました！