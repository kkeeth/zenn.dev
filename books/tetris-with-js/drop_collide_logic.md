---
title: 'ステップ3: ゲームの落下と更新，衝突判定，ゲームオーバー処理'
---

# 3.1 ゲームの状態を管理

続いてゲームの各ロジックを実装します．テトリスゲームに必要な行列（マトリックス）を生成し，ゲームエリアとプレイヤーの情報を設定するためのものです．以下に詳細を解説します．

```js
/**
 * @param {number} w - 行列の幅（列数）
 * @param {number} h - 行列の高さ（行数）
 */
const createMatrix = (w, h) => {
  const matrix = [];
  while (h--) {
    matrix.push(new Array(w).fill(0));
  }
  return matrix;
};
```

ここでは、マトリックスの生成を司る関数を定義していますが，関数内の `while` で，`h` が 0 になるまでループを実行し，ループの各回で次の操作を行っています．

`matrix.push(new Array(w).fill(0))` は，幅 `w` の配列を作成し，その配列の全ての要素を 0 で埋めた後，その配列を `matrix` という配列に追加します．これにより，`matrix` には高さ `h`，幅 `w` の 2 次元配列が生成されます．

```js
const arena = createMatrix(12, 20);
```

この行では，`createMatrix` 関数をコールして，幅 12・高さ 20 のマトリックスを作成しています．これがテトリスゲームのプレイフィールドとなります．

```js
const player = {
  pos: { x: 0, y: 0 },
  matrix: createPiece('T'), // ※ダミーで設定．後に書き換えます
  score: 0,
};
```

この `player` オブジェクトは文字通りプレイヤーに関する情報を持っています．

- `pos`: プレイヤーの現在の位置を示すオブジェクト．x と y のプロパティを持ち，初期位置は `(0, 0)`．
- `matrix`: プレイヤーが操作する現在のピースの形状を表す行列．初期値は `null`．
- `score`: プレイヤーの現在のスコア．初期値は 0．

:::details `arena` 配列のイメージ
マトリックス（二次元配列）で扱いますが，行ごとに区切られたイメージの方がコードとしては良いかもしれません．

実態としては以下のようになります．

![arena 配列のイメージ](https://storage.googleapis.com/zenn-user-upload/f7b3dd7e510b-20240708.jpeg)
:::

# 3.2 ゲームのループを作成

では，ここでピースを落下させてみましょう．そのためには，ゲームのフレームの更新と再描画を行うループを作成する必要があります．まずは今後使うための変数を定義します．

```js
let dropCounter = 0;
let dropInterval = 1000;
let lastTime = 0;
let animationId;
```

- `dropCounter`: ピースが落下するまでの経過時間を追跡するためのカウンター
- `dropInterval`: ピースが落下するまでの時間間隔（ミリ秒単位）．ここでは 1 秒（1000 ミリ秒）を設定
- `lastTime`: 前回のフレームが更新された時間を保持する変数
- `animationId`: `requestAnimationFrame`[^1] で返される ID．アニメーションフレームの更新を制御するために使用

```js
/**
 * @param {number} time - requestAnimationFrameから渡されるタイムスタンプ
 */
const update = (time = 0) => {
  // フレーム間の経過時間の計算
  const deltaTime = time - lastTime;
  lastTime = time;

  // ブロックの落下処理
  dropCounter += deltaTime;
  if (dropCounter > dropInterval) {
    playerDrop();
  }

  draw();
  animationId = requestAnimationFrame(update);
};
```

アニメーションフレームの更新を行う `update` 関数を定義しています．`deltaTime` は前回のフレームからの経過時間を計算して格納，その後，`lastTime` 変数を現在の `time` で更新します．

計算処理後，`dropCounter` に経過時間を加算します．`dropCounter` が `dropInterval` を超えた場合，`playerDrop` 関数を呼び出してピースを落下させます．こうすることで，ピースは一定間隔で落下しますが，左右に移動させたり回転させてもリアルタイムに画面には反映させられるようにできます．

最後に `draw` 関数をコールしてゲームを描画し，その後 `requestAnimationFrame` をコールして，次のフレームの更新をスケジュールします．これにより，`update` 関数が再び呼び出されます．

:::details `setInterval`
単にループをするなら `setInterval` という組み込み関数もありますが，いくつかの点を考慮したうえで `requestAnimationFrame` の方がアニメーションやゲームのループに適していると判断したので，今回はこちらを採用しています．
:::

# 3.3 ピースの落下を実装

ゲームの更新処理は書きましたが，ピースの落下処理を司る `playDrop` 関数はまだ実装されていないので作っていきましょう．以下を追記してください．

```js
const playerDrop = () => {
  player.pos.y++;
  dropCounter = 0;
};
```

ピースの位置を表す `player.pos` オブジェクトの `y` プロパティをインクリメントし，１つ位置を下にズラして描画することで落下を表現します．落下後はカウンターを初期化する必要があるため `dropCounter` を 0 に設定します．

落下したピースを表示するため `draw` 関数に以下を追記します．

```diff
  function draw() {
    context.fillStyle = '#000';
    context.fillRect(0, 0, canvas.width, canvas.height);

+   drawMatrix(arena, {x: 0, y: 0});
+   drawMatrix(player.matrix, player.pos);
  }
```

では実際に画面で確認してみたいので，`app.js` の一番下を以下のように変更してください．そうすると，下の画像のように，`T` のブロックが 1 秒毎に下に落下していくと思います．

```diff
- drawMatrix(createPiece('T'), { x: 5, y: 5 });
+ update()
```

![ピースが落下](https://storage.googleapis.com/zenn-user-upload/c5b7ae2aa3b2-20240708.png)

# 3.4 衝突判定の実装

現状はピースが画面の一番下まで移動しても，1 秒後にそのまま画面の枠外まで落下してしまいます．これはまだ衝突判定が実装されていないためですので，ピースが他のピースや壁と衝突するかどうかを判定する関数 `collide` を作成します．

```js
/**
 * @param {object} arena - ゲームフィールドを表す2次元配列
 * @param {object} player - プレイヤーの情報を持つオブジェクトで，現在のピースの形状（matrix）と位置（pos）を含む
 */
const collide = (arena, player) => {
  const [m, o] = [player.matrix, player.pos];
  for (let y = 0; y < m.length; ++y) {
    for (let x = 0; x < m[y].length; ++x) {
      if (m[y][x] !== 0 && (arena[y + o.y] && arena[y + o.y][x + o.x]) !== 0) {
        if (y + o.y <= 0) {
          return true;
        }
        return true;
      }
    }
  }
  return false;
};
```

細かく解説します．

```js
const [m, o] = [player.matrix, player.pos];
```

分割代入（destructuring assignment）を用いて，プレイヤーのピースの形状を `m`，ピースの位置を `o` に割り当てています．これは単に毎回アクセスするコストが高いため，変数に持つようしています．

続いて，衝突判定のループですが，まず外側のループでピースの行を１つずつ処理し，内側のループでピースの各行内のセルを 1 つずつ処理しています．

```js
if (m[y][x] !== 0 &&
  (arena[y + o.y] &&
  arena[y + o.y][x + o.x]) !== 0) {
```

ここの記述が，衝突判定のコアロジックとなります．

- `m[y][x] !== 0`: ピースの現在のセルが空でないことを確認
- `(arena[y + o.y] && arena[y + o.y][x + o.x]) !== 0`: ゲームフィールドの対応するセルが空でないことを確認

この条件が真（`truthy`）であれば，プレイヤーのブロックがゲームフィールドと衝突していることを意味します．

```js
if (y + o.y <= 0) {
  return true;
}
```

最後に，スタートの `y + o.y` の値が 0 の場合，上端を意味するので，上部への到達（つまり衝突）の判定となります．

# 3.5 ピースのリセットとゲームオーバー

上端に達したり他のピースゲームオーバーとなりますので，ピースのリセットとゲームオーバーの判定および処理を実装します．

```js
const playerReset = () => {
  // ピースの全種類
  const pieces = 'TJLOSZI';
  player.matrix = createPiece(pieces[(pieces.length * Math.random()) | 0]);
  player.pos.y = 0;
  player.pos.x =
    ((arena[0].length / 2) | 0) - ((player.matrix[0].length / 2) | 0);
  if (collide(arena, player)) {
    // ゲームオーバー
    cancelAnimationFrame(animationId);
  }
};
```

`pieces.length * Math.random() | 0` は，`pieces` の長さに基づいてランダムなインデックスを生成します．`Math.random()` は 0 以上 1 未満のランダムな小数を生成し，それに `pieces.length` を掛けた後，ビット論理積演算子 `|` を使用して整数に変換します．

```js
player.pos.y = 0;
player.pos.x =
  ((arena[0].length / 2) | 0) - ((player.matrix[0].length / 2) | 0);
```

の部分ですが，`player.pos.y = 0` は、見た通りプレイヤーの Y 座標を 0 に初期化します（画面の上端に配置）．`player.pos.x` の計算処理は，プレイヤーの X 座標をゲームフィールドの中央に初期化しています．これは，フィールドの幅の半分から今から落下するピースの幅の半分を引いた位置になります。

```js
if (collide(arena, player)) {
  // ゲームオーバー
  cancelAnimationFrame(animationId);
}
```

続いて `collide` 関数を用いて衝突判定を行い，レスポンスが `true`（つまり衝突したという判定）の場合はゲームオーバーのため，アニメーションを停止します．`cancelAnimationFrame(animationId)` は，`requestAnimationFrame` によってスケジュールされたアニメーションフレームの更新を停止します。これにより、ゲームループが停止します．

# 3.6 フィールドの配列の更新

現時点でどのような挙動になっているか確認してみましょう．`app.js` の `playerDrop` 関数に以下追記してください．

```diff
const playerDrop = () => {
  player.pos.y++;

+ if (collide(arena, player)) {
+   player.pos.y--;
+   playerReset();
+ }

  dropCounter = 0;
}
```

実行してみますと，以下のように画面の下端に到達した時しっかりピースが止まるようにはなりましたが，また新しくゲームがスタートしてしまっています．

![現時点でのテトリスの状況](https://storage.googleapis.com/zenn-user-upload/f3acb9716866-20240708.gif)

これはフィールドの状態を管理する `arena` 変数に一番下まで落下したピースの情報が格納されていないからですので，直しましょう．今回は `merge` という関数名で実装します．

```js
const merge = (arena, player) => {
  player.matrix.forEach((row, y) => {
    row.forEach((value, x) => {
      if (value !== 0) {
        arena[y + player.pos.y][x + player.pos.x] = value;
      }
    });
  });
};
```

二重の `forEach` 文は今まで何度か出てきたときと同様です．マトリックス（二次元配列）を扱っており，セルごとに処理をするためとなります．`row` が Y 座標，`x` が X 座標，`value` がピースの実際の値となります．

`value` が 0 でない，つまり現在のセルが空ではない場合，`arena[y + player.pos.y][x + player.pos.x] = value` の部分で値を配列に格納することで，ブロックのセルをフィールドの対応する位置に固定します．これにより，ブロックがフィールドに追加されます．

では実際に見てみましょう．`playerDrop` 関数に以下を追記してください．
※ついでにゲームスタート時点でリセットし直しております．

```diff
  if (collide(arena, player)) {
    player.pos.y--;
+   merge(arena, player);
    playerReset();
  }

//（app.jsの末尾）

+ playerReset();
  update();
```

以下のように，しっかり落下し終わったピースが残ったまま次のピースの落下が始まっていると思います．画像では `arena` 配列の中身もブラウザのコンソールに吐き出してみましたが，しっかり格納されていることが確認できます．

![arena に落下したピースの情報が格納されていることの確認](https://storage.googleapis.com/zenn-user-upload/52ab1a83a134-20240708.png)
