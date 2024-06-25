---
title: "ステップ4: ゲームの更新と描画"
---

# 4.1 ゲームのループを作成

ゲームの更新と描画を行うループを作成します．

```js
let dropCounter = 0;
let dropInterval = 1000;
let lastTime = 0;
let animationId;

// （中略）

const update = (time = 0) => {
  const deltaTime = time - lastTime;
  lastTime = time;

  dropCounter += deltaTime;
  if (dropCounter > dropInterval) {
    playerDrop();
  }

  draw();
  animationId = requestAnimationFrame(update);
}
```

落下したピースを表示するため `draw` 関数に以下を追記します．

```diff
  function draw() {
      context.fillStyle = '#000';
      context.fillRect(0, 0, canvas.width, canvas.height);

+     drawMatrix(arena, {x: 0, y: 0});
+     drawMatrix(player.matrix, player.pos);
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