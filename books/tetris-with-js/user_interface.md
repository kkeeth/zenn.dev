---
title: "ステップ5: ユーザーインターフェース"
---

# 5.1 スコアの更新

スコアを表示するためのロジックを実装します．

```js
const updateScore = () => {
  document.querySelector('#score').innerText = player.score;
}
```

# 5.2 キーボード入力

キーボード入力でピースを操作するためのロジックを実装します．

```js
document.addEventListener('keydown', event => {
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
```

# 5.3 リスタートボタン

ゲームをリスタートするためのボタンを実装します．

```js
const restartGame = () => {
  arena.forEach(row => row.fill(0));
  player.score = 0;
  updateScore();
  playerReset();
  lastTime = 0;  // Reset lastTime for the animation frame
  update();
}

document.getElementById('restartButton').addEventListener('click', restartGame);
```

# 5.4 実行

では最後に，実行していきましょう．

```js
playerReset();
updateScore();
update();
```