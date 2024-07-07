---
title: "ステップ1: プロジェクトの準備"
---

# 1.1 必要なファイルを作成

任意の場所にプロジェクト用のフォルダを作成し，以下のファイルを作成します．

```
コードをコピーする
tetris/
├── index.html
├── styles.css
└── app.js
```

# 1.2 マークアップとスタイリング

まずは `index.html` に基本的な HTML を記述します．

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>JavaScript でテトリス</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <div id="scoreBoard">Score: <span id="score">0</span></div>
  <!-- サイズを大きくしたい場合は，個々の値を調整してください-->
  <canvas id="tetris" width="240" height="400"></canvas>
  <button id="restartButton">Restart</button>
  <script src="app.js"></script>
</body>
</html>
```

続いて `styles.css` に基本的なスタイリングを設定します．

```css
body {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    height: 100vh;
    margin: 0;
    background: #000;
}

canvas {
    border: 1px solid #fff;
}

#scoreBoard {
    color: #fff;
    font-size: 20px;
    margin-bottom: 10px;
}

#restartButton {
    display: block;
    margin-top: 20px;
    padding: 10px 20px;
    font-size: 20px;
}
```

以上で HTML と CSS の設定は終わりです！

# 1.3 JavaScript の記述

ここからはひたすら JavaScript でロジックを書いていきます💁

今回は `Canvas（以下，キャンバス）` と `Canvas API` を利用して作成するため，まずはキャンバスの初期設定を行います．`app.js` に以下を追記してください．

```js
const canvas = document.querySelector('#tetris');
const context = canvas.getContext('2d');
context.scale(20, 20);
```

`context.scale(x, y)` メソッドは，キャンバスの描画スケールを指定された倍率に設定します．今回は X 軸と Y 軸の両方に対して20倍のスケールが適用されます．これにより，後続の描画操作は，元のサイズの20倍に拡大されます．

テトリスは以下のように画面の特定の領域を格子状に分割しており，今回は１マスのサイズを 20×20 とし，横12マス，縦20マスの領域に分割しています．

![枠の分割](https://storage.googleapis.com/zenn-user-upload/30c54bcf12c1-20240707.jpeg)

# 1.4 描画関数の実装

キャンバスに色を塗る関数を追加します．

```js
const draw = () => {
  context.fillStyle = '#000';
  context.fillRect(0, 0, canvas.width, canvas.height);
}
```

ここまでかけましたら，`index.html` をブラウザで開いてください．以下のような表示になったら完成です．

![テトリスの雛形](https://storage.googleapis.com/zenn-user-upload/6e8a06a15f5c-20240625.png)