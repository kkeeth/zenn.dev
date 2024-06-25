---
title: "ステップ１：プロジェクトの準備"
---

# 1.1 必要なファイルを作成

任意の場所にプロジェクト用のフォルダを作成し，以下のファイルを作成します．

```
コードをコピーする
tetris/
│
├── index.html
├── styles.css
└── tetris.js
```

# 1.2 マークアップとスタイリング

まずは `index.html` に基本的な HTML を記述します．

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Tetris</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <div id="scoreBoard">Score: <span id="score">0</span></div>
  <!-- サイズを大きくしたい場合は，個々の値を調整してください-->
  <canvas id="tetris" width="240" height="400"></canvas>
  <button id="restartButton">Restart</button>
  <script src="tetris.js"></script>
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

ここまでかけましたら，`index.html` をブラウザで開いてください．以下のような表示になったら完成です．

![テトリスの雛形](https://storage.googleapis.com/zenn-user-upload/6e8a06a15f5c-20240625.png)