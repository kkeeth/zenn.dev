---
title: "@riotjs/ssr を express ではなく hono と組み合わせてみた"
emoji: "✊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["riot", "riotjs", "hono"]
published: false
---

こんにちは．[Keeth](https://twitter.com/kuwahara_jsri) こと桑原です．

本記事は Riot.js Advent Calendar 2023 の第21日目になります！

https://qiita.com/advent-calendar/2023/riotjs

はじめに．[Riot.js（以下、riot）](https://riot.js.org/)は，非常にシンプルかつ軽量で入門の敷居も低く，とても書きやすいコンポーネント指向のUI ライブラリです．では今回も今年の riot をいろんな視点で振り返っていきたいと思います！