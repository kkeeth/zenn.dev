---
title: "ステップ4: ピースの操作"
---

ここからは，ピースを左右に移動させたり，回転させたりなど，操作の処理を実装します💁

# 4.1 ピースの移動

まずはピースを左右に移動するロジックを実装します．

```js
/**
 * @param {number} dir - 1（右） or -1（左）
 */
const playerMove = (dir) => {
  player.pos.x += dir;

  // 衝突判定
  if (collide(arena, player)) {
    player.pos.x -= dir;
  }
}
```

左右に移動させるため，`player.pos` の `x` のみ変化させれば良いですね．また，フィールドの両端に到達して，それよりさらに外側（例：右端でさらに右）に移動させようとした場合，枠からはみ出してしまうため `-= dir` で逆方向に移動させます．これにより，超高速で元の位置に戻すことで，人間の見かけ上は移動していないようになります．

画面上の反映は後回しで，4.2 の回転の実装後に反映させます．


# 4.2 ピースの回転

続いてピースを回転するロジックを実装します．

```js
/**
 * @param {object} arena - ゲームフィールドを表す2次元配列
 * @param {object} player - プレイヤーの情報を持つオブジェクトで，現在のピースの形状（matrix）と位置（pos）を含む
 */
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
```

ちょっと数学的なお話になってしまうので詳細説明は割愛しますが，今回扱っているマトリックスは二次元正方行列のため，これを90度回転させるには

1. 行と列の転置
2. 配列の反転
   * 時計回りの場合: X軸方向（横方向）に反転のため，行ごとに抽出して `reverse` させる
   * 反時計回りの場合: Y軸方向（縦方向）に反転のため，`matrix` そのものを `reverse` させる

という処理になります．
続いて，上記で定義した `rotate` 関数をコールしてピースの回転を実施する処理を `playerRotate` という関数名で実装します．


```js
/**
 * @param {number} dir - 1（右） or -1（左）
 */
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

ステップごとに見ていきましょう．まずは初期位置とオフセットの設定です．

```js
const pos = player.pos.x;
let offset = 1;
```

* `pos`: プレイヤーの現在のX座標を保存．この位置は，衝突が解決できなかった場合に元に戻すために使用する
* `offset`: 位置調整のためのオフセット

:::message
**オフセット** は，プレイヤーのピースが回転した際に他のブロックや壁と衝突しないように位置を調整するための補正量として使われます．
:::

続いてブロックの回転の処理になります．

```js
rotate(player.matrix, dir);
```

ここで 先程の `rotate` 関数をコールし，プレイヤーのピースを指定された方向に回転させます．`dir` が正（`1`）なら時計回り，負（`-1`）なら反時計回りに回転します．

続いて，回転後に衝突する場合がありますので，その解決処理になります．

```js
while (collide(arena, player)) {
  player.pos.x += offset;
  offset = -(offset + (offset > 0 ? 1 : -1));
  if (offset > player.matrix[0].length) {
    rotate(player.matrix, -dir);
    player.pos.x = pos;
    return;
  }
}
```

* `while (collide(arena, player))`: `collide` 関数をコールして．回転後のピースが他のピースや壁と衝突しているかを確認．衝突が検出される限り，次の処理を繰り返す
  * `player.pos.x += offset`: プレイヤーのX座標をオフセット量だけ移動
  * `offset = -(offset + (offset > 0 ? 1 : -1))`: オフセットを反転させ，次の位置調整のためのオフセット量を増加させる．これにより，位置調整の試行が左右交互に広がっていく形
  * `if (offset > player.matrix[0].length)`: オフセットがピースの幅を超えた場合，**位置調整がうまくいかなかった，つまり回転できない場所であることを意味** する
    * `rotate(player.matrix, -dir)` で、ピースを元の方向に回転させる（回転を元に戻す）
    * `player.pos.x = pos` で，X座標を元の位置に戻す

# 4.3 実際に画面に反映

では，実際に画面に反映させて，ピースを移動・回転させてみましょう．`app.js` に以下を追記してください．

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

**操作はキーボードを想定** しています．キーボードのキーが押されたときに特定の関数を実行するイベントリスナーを設定します．`keydown` イベントは，キーボードのキーが押された瞬間に発生します．

各 `keyCode` の値とキーボードのキーとの対照は以下です．

* `37`: 左矢印キー
* `39`: 右矢印キー
* `40`: 下矢印キー
* `81`: `Q` キー（反時計回り）
* `87`: `W` キー（時計回り）

※ついでに，十字キーの下を押下したときは１つ下に落下させるようにしています．

ここまでの実装で，ピースの移動，回転まで完了しました．

# 4.4 ラインの消去

最後に，ゲームフィールド（アリーナ）内の完全に埋まった行を検出し，消去するロジックを実装します．これがないと点数が加算されませんからね（笑）

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

ゲームフィールド（アリーナ）内の完全に埋まった行を削除後，上の行を下に移動させる処理を行っています．また，行が削除されるごとにスコアを更新します．

こちらもステップごとに見ていきましょう．まずは外側のループ（行のチェック）です．

```js
outer: for (let y = arena.length - 1; y > 0; --y) {
```

今回も２重 for 文を使っていますが，その外側について．外側の for ループは，ゲームフィールドの下から上に向かって各行を反復処理します．`y` は現在の行のインデックスです．`outer:` は単なるラベルで，内側のループから外側のループに対して `continue` 文を使うために使います．

続いて内側のループ（列のチェック）です．

```js
for (let x = 0; x < arena[y].length; ++x) {
  if (arena[y][x] === 0) {
    continue outer;
  }
}
```

内側の for ループは，現在の行（`y`）の各セルを反復処理します．`x` は現在の列のインデックスです．`if (arena[y][x] === 0)` は，現在のセルが空（`0`）であるかどうかをチェックします．

現在の行に１つでも空のセルがある場合，完全に埋まった行ではないので，`continue outer` を使って外側のループに戻ります．これにより，次の行のチェックに進みます．


最後に，行の削除と挿入です．

```js
const row = arena.splice(y, 1)[0].fill(0);
arena.unshift(row);
++y;
```

`arena.splice(y, 1)` で，現在の行（`y`）をゲームフィールドから削除します．ただし，`splice` メソッドは削除された要素を含む配列を返すため，続く `[0].fill(0)` では，削除された行の配列を 0 で埋めることになります．

`arena.unshift(row)` は、0 で埋められた行をゲームフィールドの一番上に追加します．これにより，削除された行の上のすべての行が１行下に移動します。

`++y` は，行を削除した後に行インデックスを１増やします．これにより，削除された行の新しい位置で同じインデックスを再チェックできます．

ここまででロジックは一通り実装し終わりました！実際にラインを完全に埋めると，行が削除されることが分かると思います！