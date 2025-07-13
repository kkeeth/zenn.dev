---
title: 'ステップ5: ユーザーインターフェース'
---

# 5.1 スコアの更新

では最後に，スコアを表示するためのロジックを実装します．`app.js` に以下を追記してください．

```js
const updateScore = () => {
  document.querySelector('#score').innerText = player.score;
};
```

スコアの更新を画面に反映させましょう．

```diff
  const playerDrop = () => {
    player.pos.y++;
    if (collide(arena, player)) {
      player.pos.y--;
      merge(arena, player);
      playerReset();
+     arenaSweep();
+     updateScore();
    }
    dropCounter = 0;
  }

  const restartGame = () => {
    arena.forEach(row => row.fill(0));
    player.score = 0;
+   updateScore();
    playerReset();
    lastTime = 0;  // Reset lastTime for the animation frame
    update();
  }
```

ここまでで，ゲームとしてはほぼ完成です 💁

# 5.3 リスタートボタン

ゲームオーバー後，ゲームをリスタートするためのボタンクリック時の処理を実装します．

※厳密には，いつでもボタンを押してリスタートできます

```js
const restartGame = () => {
  arena.forEach((row) => row.fill(0));
  player.score = 0;
  updateScore();
  playerReset();
  lastTime = 0;
  update();
};

document.getElementById('restartButton').addEventListener('click', restartGame);
```

今まで実装してきた初期化の処理を，`restartGame` 関数内でもう１回行っています．

以上ですべての実装が完了です！一応完成版のコードも置いておきます 💁

# 完成版のコード

参考に，完成版の JavaScript コードを載せておきます．

:::details 完成コード

```js
const canvas = document.querySelector('#tetris');
const context = canvas.getContext('2d');
let dropCounter = 0;
let dropInterval = 1000;
let animationId;
let lastTime = 0;
let isGameOver = false;

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

context.scale(20, 20);

const draw = () => {
  context.fillStyle = '#000';
  context.fillRect(0, 0, canvas.width, canvas.height);

  drawMatrix(arena, { x: 0, y: 0 });
  drawMatrix(player.matrix, player.pos);
};

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
};

const drawMatrix = (matrix, offset) => {
  matrix.forEach((row, y) => {
    row.forEach((value, x) => {
      if (value !== 0) {
        context.fillStyle = colors[value];
        context.fillRect(x + offset.x, y + offset.y, 1, 1);
      }
    });
  });
};

const createMatrix = (w, h) => {
  const matrix = [];
  while (h--) {
    matrix.push(new Array(w).fill(0));
  }
  return matrix;
};

const player = {
  pos: { x: 0, y: 0 },
  matrix: null,
  score: 0,
};

const arena = createMatrix(12, 20);

const collide = (arena, player) => {
  const [m, o] = [player.matrix, player.pos];
  for (let y = 0; y < m.length; ++y) {
    for (let x = 0; x < m[y].length; ++x) {
      if (m[y][x] !== 0 && (arena[y + o.y] && arena[y + o.y][x + o.x]) !== 0) {
        // if (y + o.y <= 0) {
        //   return true; // 荳企Κ縺ｫ蛻ｰ驕�
        // }
        return true;
      }
    }
  }
  return false;
};

const playerReset = () => {
  const pieces = 'TJLOSZI';
  player.matrix = createPiece(pieces[(pieces.length * Math.random()) | 0]);
  player.pos.y = 0;
  player.pos.x =
    ((arena[0].length / 2) | 0) - ((player.matrix[0].length / 2) | 0);
  if (collide(arena, player)) {
    gameOver();
  }
};

const gameOver = () => {
  isGameOver = true;
  if (animationId) {
    cancelAnimationFrame(animationId);
    animationId = null;
  }
  const restartButton = document.getElementById('restartButton');
  if (restartButton) {
    restartButton.style.display = 'block';
  }
};

const playerMove = (dir) => {
  player.pos.x += dir;
  if (collide(arena, player)) {
    player.pos.x -= dir;
  }
};

const rotate = (matrix, dir) => {
  for (let y = 0; y < matrix.length; ++y) {
    for (let x = 0; x < y; ++x) {
      [matrix[x][y], matrix[y][x]] = [matrix[y][x], matrix[x][y]];
    }
  }

  if (dir > 0) {
    matrix.forEach((row) => row.reverse());
  } else {
    matrix.reverse();
  }
};

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
};

const update = (time = 0) => {
  if (isGameOver) {
    return;
  }
  const deltaTime = time - lastTime;
  lastTime = time;

  dropCounter += deltaTime;
  if (dropCounter > dropInterval) {
    playerDrop();
  }

  draw();
  animationId = requestAnimationFrame(update);
};

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
};

const merge = (arena, player) => {
  player.matrix.forEach((row, y) => {
    row.forEach((value, x) => {
      if (value !== 0) {
        arena[y + player.pos.y][x + player.pos.x] = value;
      }
    });
  });
};

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
};

const updateScore = () => {
  const scoreElement = document.querySelector('#score');
  if (scoreElement) {
    scoreElement.innerText = player.score;
  }
};

document.addEventListener('keydown', (event) => {
  if (isGameOver) {
    return;
  }
  if (event.keyCode === 37) {
    playerMove(-1);
  } else if (event.keyCode === 39) {
    playerMove(1);
  } else if (event.keyCode === 40) {
    playerDrop();
  } else if (event.keyCode === 81) {
    playerRotate(-1);
  } else if (event.keyCode === 87) {
    playerRotate(1);
  }
});

const restartGame = () => {
  isGameOver = false;
  if (animationId) {
    cancelAnimationFrame(animationId);
    animationId = null;
  }
  arena.forEach((row) => row.fill(0));
  player.score = 0;
  updateScore();
  playerReset();
  lastTime = 0; // Reset lastTime for the animation frame

  const restartButton = document.getElementById('restartButton');
  if (restartButton) {
    restartButton.style.display = 'none';
  }

  update();
};

const restartButton = document.getElementById('restartButton');
if (restartButton) {
  restartButton.addEventListener('click', restartGame);
}

playerReset();
updateScore();
update();
```

:::

以上！ではでは(=ﾟ ω ﾟ)ﾉ
