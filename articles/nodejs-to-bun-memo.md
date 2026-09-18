---
title: 'Node.js から Bun へ移行する際に知っておきたいこと【備忘録】'
emoji: '🍞'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['bun', 'nodejs', 'npm', 'pnpm', 'javascript']
published: false
---

こんにちは．[Keeth](https://x.com/kuwahara_jsri) こと桑原です．

手元のプロジェクトを Node.js から [Bun](https://bun.sh/) に置き換えようとして，パッケージマネージャの違いやロックファイルの形式，pnpm の symlink 構造との比較あたりを一度整理しておきたくなりました．調べたことをそのまま備忘録として残します．自分が後で読み返すためのメモなので，表と箇条書きが多めです 🙏

※ 間違っている記述があればバンバンコメントいただけると嬉しいです 🙇‍♂️

## TL;DR

- 移行は段階的に．パッケージマネージャだけ Bun に替えるところから始めて，ランタイムの置き換えは最後に回すのが安全
- `bun.lock`（テキストの JSONC 形式）は v1.2 以降のデフォルト．`bun.lockb` と違って git diff が読めるので，既存プロジェクトは変換しておきたい
- ネイティブアドオンとレガシーな CLI ツール以外は，Bun 単体でだいたい完結する
- pnpm は symlink と hardlink でディスク効率が最強．ただし Docker 環境では恩恵が薄い
- bun はフラット構造 + hardlink で，互換性と速度を取っている．phantom dependencies は防げない

## 1. 既存の Node.js プロジェクトを Bun に置き換える手順

### 1-1. Bun のインストール

```bash
curl -fsSL https://bun.sh/install | bash
```

### 1-2. 既存の `node_modules` とロックファイルを削除

```bash
rm -rf node_modules package-lock.json yarn.lock pnpm-lock.yaml
```

### 1-3. Bun で依存関係を再インストール

```bash
bun install
```

`bun.lockb`（バイナリ）または `bun.lock`（テキスト，v1.2 以降）が生成されます．

### 1-4. `package.json` の scripts を書き換え

```json
{
  "scripts": {
    "dev": "bun run --hot src/index.ts",
    "start": "bun run src/index.ts",
    "test": "bun test",
    "build": "bun build src/index.ts --outdir ./dist"
  }
}
```

置き換えの対応はこうなります．

- `node` → `bun run`（または直接 `bun`）
- `nodemon` / `ts-node` / `tsx` → `bun --hot` があるので不要
- `jest` / `vitest` → `bun test` に置換可能（API 互換あり）
- `npm` / `yarn` / `pnpm` → `bun`

### 1-5. API の差分を確認

Bun は大半の Node.js API と互換ですが，以下は注意が必要です．

- `fs`，`path`，`http` などは動きますが，`Bun.file()` や `Bun.serve()`，`Bun.write()` のほうが高速です
- `process.env` はそのまま使えて，さらに `.env` を自動でロードしてくれます（dotenv 不要）
- TypeScript と JSX / TSX はトランスパイルなしでそのまま実行できます
- ネイティブアドオン（`.node` ファイル）は一部非対応です

### 1-6. CI や Dockerfile を更新

```dockerfile
FROM oven/bun:1
WORKDIR /app
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile
COPY . .
CMD ["bun", "run", "start"]
```

### 1-7. 動作確認

```bash
bun run dev
bun test
```

### 段階的移行のコツ

- まず `npm` を `bun` に置き換えるだけでも，インストールの高速化という恩恵があります
- ランタイムの置換（`node` → `bun`）は最後に回します．Express や Fastify などは大抵そのまま動きます
- 互換性のチェックは公式のステータスページが早いです

https://bun.sh/docs/runtime/nodejs-apis

## 2. npm と Bun のパッケージ管理の違い

### レジストリと互換性

- **同じ npm レジストリを使う**ので，`npmjs.com` の全パッケージがそのまま使えます
- `package.json` のフォーマットも完全互換です
- `node_modules` の構造も npm と同じです（pnpm のような symlink 構造ではありません）

### ロックファイル

| ツール | ロックファイル | 形式 |
| --- | --- | --- |
| npm | `package-lock.json` | JSON |
| yarn | `yarn.lock` | YAML 風 |
| pnpm | `pnpm-lock.yaml` | YAML |
| **bun** | `bun.lock`（v1.2 以降）または `bun.lockb` | テキスト JSONC / バイナリ |

`bun.lockb` はバイナリなので高速ですが，git diff が読めません．v1.2 以降はテキストの `bun.lock` がデフォルトで，こちらが推奨です．

### コマンド対応表

| npm | bun |
| --- | --- |
| `npm install` | `bun install`（`bun i`） |
| `npm install <pkg>` | `bun add <pkg>` |
| `npm install -D <pkg>` | `bun add -d <pkg>` |
| `npm install -g <pkg>` | `bun add -g <pkg>` |
| `npm uninstall <pkg>` | `bun remove <pkg>` |
| `npm update` | `bun update` |
| `npm run <script>` | `bun run <script>`（または `bun <script>`） |
| `npm exec` / `npx` | `bunx` |
| `npm ci` | `bun install --frozen-lockfile` |
| `npm publish` | `bun publish` |
| `npm outdated` | `bun outdated` |
| `npm pack` | `bun pm pack` |

### インストール速度

- npm の **20 〜 30 倍高速**とされています（グローバルキャッシュ，並列ダウンロード，hardlink の組み合わせ）
- キャッシュは `~/.bun/install/cache` に集約されます

### Workspaces（モノレポ）

- npm や yarn の workspaces と互換で，`package.json` の `workspaces` フィールドをそのまま読みます
- `bun install` でワークスペース全体を一括インストールできます
- `bun --filter <workspace> <script>` でフィルタ実行ができます

### バージョン管理

`bun pm` のサブコマンドで詳細を管理します．

- `bun pm ls` で依存ツリーを表示
- `bun pm cache` でキャッシュを操作
- `bun pm trust` で postinstall スクリプトの許可を管理

### エコシステムの違い（ビルトイン機能）

| 機能 | Node.js での定番 | Bun では |
| --- | --- | --- |
| TypeScript 実行 | `ts-node` / `tsx` | **ビルトイン** |
| JSX / TSX 実行 | Babel / esbuild | **ビルトイン** |
| ホットリロード | `nodemon` | `bun --hot` |
| 環境変数の読み込み | `dotenv` | **ビルトイン**（`.env` 自動） |
| バンドラ | webpack / esbuild / Vite | `bun build` |
| テストランナー | Jest / Vitest | `bun test`（Jest 互換 API） |
| パッケージ実行 | `npx` | `bunx` |
| WebSocket サーバ | `ws` | `Bun.serve({websocket})` |
| パスワードハッシュ | `bcrypt` | `Bun.password` |
| SQLite | `better-sqlite3` | `bun:sqlite` |
| Postgres | `pg` | `Bun.sql`（v1.2 以降） |
| Redis | `redis` / `ioredis` | `Bun.redis`（v1.2 以降） |
| S3 互換ストレージ | `aws-sdk` | `Bun.s3`（v1.2 以降） |
| ファイル I/O | `fs.promises` | `Bun.file()`（より高速） |

### セキュリティモデル

- **postinstall スクリプトはデフォルトで実行されません**（npm はデフォルトで実行します）
- 実行させたい場合は `trustedDependencies` を `package.json` に明示する必要があります

```json
{ "trustedDependencies": ["esbuild", "sharp"] }
```

- `bun pm trust <pkg>` で対話的に許可することもできます

### モジュール解決

- **CommonJS と ESM をシームレスに混在できます**．`require()` と `import` を同一ファイルで使えます
- Node.js より制約が緩く，`.js` 拡張子の省略も TypeScript のままの実行も通ります
- `package.json` の `type` フィールドの影響を受けにくいです

### 非互換と注意点

- ネイティブアドオン（`.node` ファイル）は **N-API 経由なら動きます**が，一部は未対応です
- `node-gyp` を使うパッケージは動作しないことがあります
- Worker Threads は対応していますが，API に細かい差異があります
- `cluster` モジュールのサポートは限定的です
- 一部の Node.js 内部 API（`process.binding` など）は非対応です

### Web 標準 API 優先

- `fetch`，`Request`，`Response`，`WebSocket`，`Blob`，`FormData`，`URL` がすべてネイティブです
- Node.js 固有の API より Web 標準の API を使う，という設計思想が全体に効いています
- `Bun.serve()` は Express より速いですが，Fetch API 準拠の書き方になります

### 移行時の落とし穴

1. **postinstall が動かない** → `trustedDependencies` に追加します
2. **`bun.lockb` の diff が読めない** → v1.2 以降で `bun install --save-text-lockfile` を実行してテキスト化します
3. **CI で `npm ci` 相当が欲しい** → `bun install --frozen-lockfile` を使います
4. **グローバル bin の PATH** → `~/.bun/bin` を PATH に追加します
5. **モノレポで peerDependencies を厳密に見たい** → `bun install --strict-peer-dependencies` で検証します

## 3. テキスト JSONC とは？Bun だけで動くのか

### JSONC とは

JSONC は JSON with Comments（コメント付き JSON）の略です．

#### 通常の JSON との違い

- `//` や `/* */` の **コメントが書けます**
- 配列やオブジェクトの末尾の **トレイリングカンマが許容されます**
- 拡張子は `.json` または `.jsonc` です
- VS Code の `settings.json` や `tsconfig.json` などで採用されている形式です

#### 例

```jsonc
{
  // 依存関係の定義
  "dependencies": {
    "react": "^18.0.0",
    "lodash": "^4.17.21",  // 末尾カンマOK
  },
  /* ブロックコメントも使える */
  "version": "1.0.0",
}
```

### Bun の `bun.lock`（テキスト形式）

v1.2 以降のデフォルトです．`bun.lockb`（バイナリ）と比べるとこうなります．

| 項目 | `bun.lockb`（バイナリ） | `bun.lock`（JSONC） |
| --- | --- | --- |
| 可読性 | 不可（バイナリ） | 人間が読める |
| git diff | 表示不可 | 通常どおり表示 |
| マージコンフリクト解消 | 困難 | 容易 |
| ファイルサイズ | 小 | 中 |
| パース速度 | 高速 | 高速（独自パーサ） |
| コードレビュー | 不可 | 可能 |

実際の `bun.lock` の中身は，こういう構造になっています．

```jsonc
{
  "lockfileVersion": 1,
  "workspaces": {
    "": {
      "name": "my-app",
      "dependencies": {
        "react": "^18.0.0"
      }
    }
  },
  "packages": {
    "react": ["react@18.2.0", "...", { /* metadata */ }, "sha512-..."]
  }
}
```

#### 既存プロジェクトでの切り替え

```bash
bun install --save-text-lockfile  # bun.lockb → bun.lock に変換
rm bun.lockb                      # 古いバイナリ版を削除
```

### Bun だけで Node.js なしで動作するのか

結論としては，ほとんどのケースで動きます．Node.js をアンインストールしても開発できました．

#### Bun 単体で完結できるもの

- TypeScript と JavaScript の実行（`bun run`）
- パッケージ管理（`bun install`）
- スクリプト実行（`bunx` が `npx` 相当）
- バンドル（`bun build`）
- テスト（`bun test`）
- 単一実行ファイル化（`bun build --compile`）
- Web サーバや API サーバの起動
- React や Next.js，Vite などのフレームワーク開発

#### Node.js が必要になるケース

**1. ネイティブアドオンに依存するパッケージ**

- `node-gyp` で C++ のビルドが必要なもの
- 古いバージョンの `canvas` や `node-sass`，一部の `bcrypt` 実装などが該当します
- 多くは代替パッケージか Bun のビルトインで置き換えられます

**2. Node.js 固有の内部 API**

```js
process.binding(...)        // 非対応
process._linkedBinding(...) // 非対応
v8.* の一部API              // 部分対応
vm モジュールの一部         // 部分対応
```

**3. `cluster` モジュールを多用するアプリ**

- Bun にも実装はありますが，成熟度は Node.js のほうが上です
- PM2 などのプロセスマネージャと併用する場合は検証が必要です

**4. Electron や一部のデスクトップアプリ**

- Electron は Node.js ランタイムを内蔵しているため，開発時に Node.js が必要になります

**5. 特定の CLI ツール**

- `node` バイナリを直接呼び出しているシェルスクリプトや Makefile
- shebang が `#!/usr/bin/env node` で固定されているもの．`#!/usr/bin/env bun` に書き換えるか，`bun run` で起動します

**6. ツールチェーン側が Node.js 前提**

- 一部の IDE プラグインやデバッガが `node` バイナリを期待します
- VS Code のデバッガは設定で `runtimeExecutable: "bun"` に変更できます

#### Node.js なしの開発環境を作ってみる

```bash
# Node.js をアンインストールして Bun だけ入れる
brew uninstall node
curl -fsSL https://bun.sh/install | bash

# 普通に開発できる
bun create vite my-app
cd my-app
bun install
bun run dev          # Vite 開発サーバが起動
bun test             # テスト実行
bun run build        # ビルド
```

#### Bun が提供する `node` 互換性

- Bun は Node.js の主要モジュール（`fs`，`path`，`http`，`crypto`，`stream` など）を内部で **再実装** しています
- `import fs from "node:fs"` のような `node:` プレフィックスもサポートしています
- 互換性はバージョンごとに向上していて，最新の Bun は Node.js 20 以降の API の大部分をカバーしています．最新の状況は[互換性ステータスのページ](https://bun.sh/docs/runtime/nodejs-apis)で確認できます

#### 推奨スタンス

- **新規プロジェクト**なら，Bun のみで始めて問題ありません
- **既存の中規模以上のプロジェクト**では，Node.js と両方入れて並行運用するのが安全です．`nvm` や `mise`，`proto` などのバージョン管理ツールで両方扱えますし，問題のあるパッケージだけ Node.js で動かすハイブリッド運用もできます
- **CI や Docker** では，公式の `oven/bun` イメージを使えば Node.js は要りません

#### Node.js を残しておくと便利な場面

- レガシーな CLI ツール（`gulp` や古い `webpack` 設定など）を時々使うとき
- チームメンバー全員が Bun に移行していない段階
- パッケージの作者として，Node.js と Bun の両方で動作確認が必要なとき

## 4. pnpm の symlink 構造と各パッケージマネージャの比較

### pnpm の symlink 構造とは

pnpm は依存パッケージを **物理的に 1 ヶ所に保存** して，各プロジェクトの `node_modules` からは **symlink（シンボリックリンク）で参照** する仕組みを採用しています．

#### 通常の npm / yarn / bun の構造（フラット）

```
my-app/
└── node_modules/
    ├── react/                    ← 実体ファイル
    ├── react-dom/                ← 実体ファイル
    ├── lodash/                   ← 実体ファイル
    ├── scheduler/                ← reactの依存(巻き上げ)
    └── ...
```

- すべての依存が `node_modules` 直下に **フラットに展開** されます
- 同じパッケージを別プロジェクトで使うと，**プロジェクトごとに重複してコピー** されます
- 直接依存していないパッケージ（`scheduler` など）も `import` できてしまいます．これが **phantom dependencies 問題**です

#### pnpm の構造（content-addressable + symlink）

```
~/.pnpm-store/                    ← グローバルストア(全プロジェクト共通)
└── v3/
    └── files/
        ├── 00/abc123...          ← ハッシュベースで保存された実ファイル
        ├── 01/def456...
        └── ...

my-app/
└── node_modules/
    ├── .pnpm/                    ← 実体置き場(中央集約)
    │   ├── react@18.2.0/
    │   │   └── node_modules/
    │   │       ├── react/        ← グローバルストアへのhardlink
    │   │       └── loose-envify/ ← reactの依存(symlink)
    │   ├── react-dom@18.2.0/
    │   │   └── node_modules/
    │   │       ├── react-dom/    ← hardlink
    │   │       ├── react/        → ../../react@18.2.0/node_modules/react (symlink)
    │   │       └── scheduler/    → ../../scheduler@0.23.0/node_modules/scheduler
    │   └── scheduler@0.23.0/
    │       └── node_modules/
    │           └── scheduler/    ← hardlink
    │
    ├── react/      → .pnpm/react@18.2.0/node_modules/react       (symlink)
    └── react-dom/  → .pnpm/react-dom@18.2.0/node_modules/react-dom (symlink)
```

ストアの場所は OS や pnpm のバージョンによって変わるので，手元のパスは `pnpm store path` で確認してください．

### 仕組みの 2 段構え

#### グローバルストアから `.pnpm/` へ（hardlink）

- ファイルの実体はディスク上に **1 回だけ保存** されます
- `.pnpm/` 内のファイルは **hardlink** なので，ディスク上は同じ inode を指す別名です
- 100 プロジェクトで同じバージョンの react を使っても，**ディスクには 1 つだけ**です

#### `.pnpm/` からトップレベルの `node_modules/` へ（symlink）

- プロジェクトの `package.json` に書いた直接依存だけが，`node_modules` 直下に **symlink で現れます**
- 間接依存は `.pnpm/` の奥に隠れていて，**トップレベルからは見えません**

### hardlink と symlink の違い

| 項目 | hardlink | symlink |
| --- | --- | --- |
| 実体 | 同じ inode（ファイルシステム上は同一） | 別ファイル．パス文字列を保持 |
| 元を消したら | 残る（参照カウントで管理） | リンク切れ |
| 跨げる FS | 同一 FS 内のみ | FS 跨ぎ OK |
| ディレクトリ対応 | 不可（通常） | 可能 |
| 容量 | ほぼ 0 | ほぼ 0 |
| pnpm での用途 | ストアから `.pnpm/`（ファイル単位） | `.pnpm/` から `node_modules/`（ディレクトリ単位） |

### 各パッケージマネージャの比較

| 項目 | npm | yarn（classic） | yarn（berry / PnP） | pnpm | bun |
| --- | --- | --- | --- | --- | --- |
| `node_modules` 構造 | フラット | フラット | なし（`.yarn/cache`） | symlink + hardlink | フラット（+ hardlink） |
| ディスク使用量 | 多 | 多 | 少 | **最少** | 中 |
| インストール速度 | 遅 | 中 | 速 | 速 | **最速** |
| phantom 依存の防止 | × | × | ◎ | ◎ | × |
| 互換性 | ◎ | ◎ | △（独自解決） | ○ | ◎ |
| モノレポ対応 | ○ | ◎ | ◎ | ◎ | ○ |

### pnpm の利点

#### 1. ディスク容量の節約が効く

- Next.js のプロジェクトが 10 個あっても，react は 1 コピーだけです
- 数十 GB あった `node_modules` が数 GB になるケースもあります

#### 2. インストールが速い

- すでにストアにあるファイルはダウンロードが不要です
- ファイルコピーではなく hardlink の作成だけなので，I/O が軽いです

#### 3. strict mode で依存の正しさが保証される

```js
// pnpm環境
import _ from "lodash";  // package.jsonに書いてないとエラー
```

- npm では間接依存が `node_modules` 直下に巻き上げられるので，書いていないパッケージも `import` できてしまいます
- これが本番で「依存先のバージョンが変わって動かなくなる」問題の原因になります
- pnpm は構造上これを防げます

#### 4. モノレポで真価を発揮する

- ワークスペース間でのバージョン整合性を保ちやすいです
- 各ワークスペースから見える依存が厳密に制御されます

### pnpm の欠点と注意点

#### 1. symlink 非対応のツールとの相性

- 一部のバンドラやツールが symlink を正しく辿れません．古い webpack や一部の Electron ビルダーなどです
- 最近はほぼ解消されていますが，レガシーな環境では `shamefully-hoist=true` で回避します

#### 2. Windows 環境での制約

- Windows の symlink は Admin 権限または開発者モードが必要です
- 最近の Windows 10 以降では問題ありませんが，古い環境では注意が必要です

#### 3. Docker イメージでのレイヤキャッシュ

- hardlink は Docker のレイヤを跨げないため，**コンテナ内では恩恵が薄い**です
- Docker 利用時は npm や yarn，bun と容量差が出にくくなります

#### 4. デバッガや IDE の解決パス

- `node_modules/react/` の実パスが `.pnpm/react@18.2.0/...` になります
- スタックトレースやブレークポイントで戸惑うことがあります

### bun との比較（なぜ bun は symlink にしないのか）

bun は **フラット構造を維持しつつ hardlink でディスクを節約する**というハイブリッド方式です．

```
~/.bun/install/cache/             ← グローバルキャッシュ
└── react@18.2.0/...

my-app/
└── node_modules/
    └── react/                    ← キャッシュからのhardlink(フラット展開)
```

- **互換性を最優先**しています．全ツールがフラット構造を期待しているので，問題が起きにくいです
- **速度も優先**しています．symlink 解決のオーバーヘッドがありません
- トレードオフとして，phantom dependencies 問題は防げず，ディスクの節約も pnpm ほどではありません

### どれを選ぶべきか

| ケース | 推奨 |
| --- | --- |
| 個人の小規模から中規模のプロジェクト | bun か pnpm |
| 大規模モノレポ | **pnpm** |
| 厳密な依存管理が欲しい | **pnpm** か yarn（PnP） |
| 速度最優先 | **bun** |
| ディスク容量がカツカツ | **pnpm** |
| レガシーなツールが多い | npm か yarn（classic） |
| Docker 中心の本番環境 | bun か npm（pnpm の恩恵が薄いため） |

## まとめ

- **Bun 移行は段階的に**．パッケージマネージャだけ Bun に替えるところから始めて，ランタイムは最後に回します
- **`bun.lock`（JSONC）は v1.2 以降の標準**です．Git フレンドリーでレビューもできます
- **Bun 単体でほぼ完結します**．ネイティブアドオンやレガシーツール以外は Node.js が要りません
- **pnpm は symlink と hardlink でディスク効率が最強**です．ただし Docker 環境では恩恵が薄いです
- **bun はフラット構造 + hardlink** で，互換性と速度のバランスを取っています

## 参考リンク

- [Bun 公式ドキュメント](https://bun.sh/)
- [Node.js API の互換性ステータス](https://bun.sh/docs/runtime/nodejs-apis)
- [pnpm 公式ドキュメント](https://pnpm.io/)
