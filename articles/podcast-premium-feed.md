---
title: '個人ポッドキャストにプレミアム配信を自前で作った設計と葛藤'
emoji: '🎙️'
type: 'tech'
topics: ['cloudflare', 'firebase', 'podcast', 'architecture']
published: false
---

## はじめに

ポッドキャスト「雨宿りとWEBの小噺」を運営しています。有料メンバーシップで限定エピソードを配信したいと思い立ったのですが、実装を進めるうちに「思ったより簡単ではない」ことに気づきました。

この記事では、技術選定の過程で何を考え、何を試し、何を捨て、最終的にどういうアーキテクチャに着地したかを、思考の変遷も含めて書きます。

---

## TL;DR

- 配信プラットフォーム Art19 の Alternate Feed（プレミアム用フィード）を使う
- 音声ファイルの URL を直接リスナーに渡すとコンテンツ保護できない
- `Cloudflare Workers` でフィードと音声の両方をプロキシし、署名付き URL + KV でアクセス制御
- 課金は `Stripe`，認証・ユーザー管理は `Firebase`、エッジでの認可は `Cloudflare KV`
- 既存サービス（`Memberful`, `Supercast` 等）を使わず自前構築した理由と、そのトレードオフ
  - **素直に Memberful, Supercast 等を使った方が楽**

---

## 前提：やりたかったこと

1. Art19 で配信しているポッドキャストに**プレミアム限定エピソード**を追加
2. 公式サイト（Riot.js SPA）で通常エピソードとプレミアムエピソードを**時系列マージ表示**
3. Podcast アプリ（Apple Podcasts, Overcast 等）でもプレミアムフィードを**購読可能に**
4. 解約したら**即時アクセス停止**

---

## Phase 1：素朴な実装 — そしてすぐ壁にぶつかる

### 最初のアイデア

Art19 には Primary Feed（通常公開）と Alternate Feed（限定用）がある。クライアント側で両方取得してマージすればいいのでは？

```
クライアント → Art19 Primary Feed（公開）
クライアント → Art19 Alternate Feed（プレミアム）
→ マージして時系列ソート → 表示
```

実装は簡単だった。`fetchMergedFeeds()` で `Promise.allSettled` を使い、片方が失敗しても表示が止まらないようにした。

### 壁 1：Art19 の Embed Player が Alternate Feed で 404

Art19 の通常エピソードは `https://art19.com/shows/.../episodes/{id}/embed` で iframe プレイヤーが使える。しかし **Alternate Feed のエピソードは同じ URL にアクセスしても 404 が返る**。

→ プレミアムエピソードは HTML5 の `<audio>` タグで直接再生するしかない。RSS の `<enclosure>` から音声 URL を取得して使うことにした。

### 壁 2：Alternate Feed の URL がクライアント JS に丸見え

Vite でビルドすると `VITE_` prefix の環境変数はバンドルに含まれる。つまり **Alternate Feed の URL が誰でも見れてしまう**。Feed URL を知っていれば、全エピソードの音声 URL も RSS から取得可能。

これではプレミアムの意味がない。

---

## Phase 2：サーバーサイドに逃がす — Firebase Cloud Functions

フィード取得をクライアントからサーバーに移した。

```
クライアント → Cloud Function（getPremiumEpisodes）→ Art19 Alternate Feed
                   ↓
           認証チェック：Firebase Auth uid → Firestore plan === 'premium'
           非プレミアムユーザーには audioUrl を空文字で返す
```

**解決したこと：** Feed URL がクライアントから消えた（`grep alternate_feeds` でバンドルを確認）。

**解決していないこと：** 返された `audioUrl`（Art19 の直 URL）をプレミアムユーザーがコピーして共有すれば、誰でもアクセスできてしまう。

---

## Phase 3：他のポッドキャストはどうしている？

ここで一度立ち止まって、他のサービスやポッドキャストの方式を調査した。

### サービス比較

| サービス           | 月額手数料   | 決済手数料                  | 方式                 | 備考                         |
| ------------------ | ------------ | --------------------------- | -------------------- | ---------------------------- |
| **Supercast**      | $0           | 取引の 5.5% + Stripe 手数料 | ユーザーごと固有 RSS | 一番スムーズだが手数料が高い |
| **Memberful**      | $25/月〜     | Stripe 手数料のみ           | ユーザーごと固有 RSS | 月額固定コストが痛い         |
| **Patreon**        | $0           | 5〜12%                      | 独自プラットフォーム | ポッドキャスト特化ではない   |
| **Apple Podcasts** | $19.99/年    | Apple 15〜30%               | Apple 独自           | Apple ユーザー限定           |
| **Spotify**        | $0           | 未公開                      | Spotify 独自         | Spotify ユーザー限定         |
| **自前構築**       | インフラ実費 | Stripe 3.6%                 | 完全カスタム         | 開発・メンテコスト           |

### 他番組の方式

- **Rebuild.fm**：Web ポータルからダウンロード（RSS ではなくブラウザ経由）
- **backspace.fm（BSM）**：プラットフォームが管理するプレミアムフィード

### 判断

個人ポッドキャストで月額 $25 や 5.5% の手数料は厳しい。Art19 を使い続ける制約もある。**自前で Supercast 相当のものを作る** 方向に舵を切った。

---

## Phase 4：Art19 の認証機能を検討 → 断念

Art19 側にもフィードの保護機能がある。サポートに問い合わせたところ、2つの方式を提案された。

### Access Token 方式

フィードと音声 URL にクエリパラメータとして共有トークンを付与：

```
https://rss.art19.com/alternate_feeds/...?token=SHARED_SECRET
https://rss.art19.com/episodes/{id}.mp3?token=SHARED_SECRET
```

**問題：** トークンは全ユーザー共通。1人がURLを共有すれば全エピソードにアクセスできる。ローテーションは可能だが、既存の Podcast アプリに登録済みの URL が無効になってしまう。

### JWT 方式

Art19 に事前に公開鍵を登録し、リクエスト時に JWT を `Authorization` ヘッダーで送る。

```
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

**致命的な問題：** `<audio>` タグや Podcast アプリは HTTP リクエストに任意のヘッダーを付与できない。つまり **JWT を使うにはどのみちプロキシが必要** になる。プロキシを立てるなら JWT の意味が薄れる（プロキシ↔Art19 間はサーバー間通信なので、他の方法でも保護できる）。

```
❌ <audio src="https://art19.com/..."> → Authorization ヘッダーを付けられない
✅ <audio src="https://my-proxy/..."> → プロキシが Art19 に Authorization 付きで中継
    → でもそれならプロキシ側で認証すれば JWT 不要
```

### 結論

どちらの方式も「プロキシなしでクライアント/Podcast アプリから直接アクセス」を実現できない。**プロキシを立てるなら、Art19 の認証機能に依存せず自前で制御した方がシンプル**。

---

## Phase 5：最終アーキテクチャ — Cloudflare Workers

### なぜ Cloudflare Workers か

| 観点               | Workers                      | Firebase Functions | AWS Lambda               |
| ------------------ | ---------------------------- | ------------------ | ------------------------ |
| コールドスタート   | なし（V8 Isolates）          | 数秒               | 数百ms〜数秒             |
| エッジ実行         | 全世界 300+ PoP              | リージョン指定     | リージョン指定           |
| 音声ストリーミング | ストリーミングレスポンス対応 | 制限あり           | API Gateway タイムアウト |
| KV ストア          | 組み込み（KV）               | Firestore（遅い）  | DynamoDB（別サービス）   |
| 無料枠             | 10万リクエスト/日            | 200万回/月         | 100万回/月               |

音声プロキシに必要な要件：

- **Range ヘッダー対応**（Podcast アプリのシーク・部分ダウンロード）
- **ストリーミング転送**（音声ファイル全体をメモリに載せない）
- **低レイテンシ認証**（毎リクエスト KV を参照）

Workers は全て満たしていた。

### R2 を使わなかった理由

当初は Art19 の音声を Cloudflare R2 にコピーしてから配信することも検討した。

```
（検討案）Art19 → R2 にコピー → R2 から署名付き URL で配信
（採用案）Art19 → Workers が直接プロキシ → 署名付き URL で配信
```

**R2 を選ばなかった理由：**

- Art19 に新エピソードが追加されたとき、R2 への同期が必要（cron? webhook?）
- Art19 側でエピソードが更新・削除されたときの同期問題
- R2 のストレージコスト（音声ファイルは大きい）
- **プロキシで十分**：Workers は Art19 からストリーミングでそのままクライアントに転送するので、メモリにも載らない

### 全体像

<!-- TODO: draw.io で作図 -->

```
┌─────────────┐     ┌──────────┐     ┌───────────────────┐
│  Stripe     │────→│ Firebase │────→│  Cloudflare KV    │
│  Webhook    │     │ Functions│     │  userToken:active  │
└─────────────┘     └──────────┘     └───────────────────┘
                         │                     ↑
                    Firestore に             Workers が
                    premiumFeedToken        毎リクエスト参照
                    を保存                      │
                         │              ┌──────┴──────┐
                         ↓              │             │
                    ┌─────────┐   /feed/:token  /audio/:id
                    │  Web    │         │             │
                    │  Client │    ┌────┴────┐  ┌────┴────┐
                    └─────────┘    │ Workers │  │ Workers │
                                   │ (Feed)  │  │ (Audio) │
                                   └────┬────┘  └────┬────┘
                                        │            │
                                   Art19 Feed   Art19 Audio
                                   (fetch+     (Range proxy)
                                    rewrite)
```

---

## 処理フロー

### 購読開始フロー

```mermaid
sequenceDiagram
    actor User
    participant Stripe
    participant Functions as Firebase Functions
    participant Firestore
    participant KV as Cloudflare KV

    User->>Stripe: Payment Link で決済
    Stripe->>Functions: checkout.session.completed webhook
    Functions->>Functions: メールから Firebase Auth ユーザー特定/作成
    Functions->>Functions: premiumFeedToken = randomUUID()
    par Firestore と KV に並列書き込み
        Functions->>Firestore: plan: premium, premiumFeedToken 保存
        Functions->>KV: PUT userToken → "active"
    end
    Functions->>Stripe: 200 OK
```

### フィード取得フロー（Podcast アプリ）

```mermaid
sequenceDiagram
    actor App as Podcast App
    participant Worker as CF Worker
    participant KV as Cloudflare KV
    participant Art19

    App->>Worker: GET /feed/{userToken}
    Worker->>KV: get(userToken)
    KV-->>Worker: "active"
    Worker->>Art19: fetch(Alternate Feed URL)
    Art19-->>Worker: RSS XML
    Worker->>Worker: enclosure URL を署名付き Worker URL に書き換え
    Worker-->>App: RSS XML（書き換え済み）
```

### 音声再生フロー

```mermaid
sequenceDiagram
    actor App as Podcast App / Browser
    participant Worker as CF Worker
    participant KV as Cloudflare KV
    participant Art19

    App->>Worker: GET /audio/{episodeId}?userToken=...&expires=...&sig=...
    Worker->>Worker: HMAC 署名検証
    Worker->>Worker: 有効期限チェック
    Worker->>KV: get(userToken)
    KV-->>Worker: "active"
    Worker->>Art19: fetch(audio URL) + Range ヘッダー転送
    Art19-->>Worker: 206 Partial Content + audio stream
    Worker-->>App: 206 + ストリーミング転送
```

### 解約フロー

```mermaid
sequenceDiagram
    actor User
    participant Stripe
    participant Functions as Firebase Functions
    participant Firestore
    participant KV as Cloudflare KV

    User->>Stripe: カスタマーポータルから解約
    Stripe->>Functions: customer.subscription.deleted webhook
    Functions->>Firestore: premiumFeedToken 取得
    par
        Functions->>Firestore: plan: free, premiumFeedToken: null
        Functions->>KV: DELETE userToken
    end
    Note over KV: 次のリクエストから即 401
```

---

## 設計で悩んだポイント

### 1. 二重認証：署名付き URL だけでは不十分

署名付き URL（HMAC-SHA256 + 有効期限）だけではダメ。URL が有効期限内に共有されたら、誰でもアクセスできる。

→ **署名 + KV の二重チェック** にした。音声リクエストごとに KV で購読状態を確認する。解約後は KV から削除するので、署名が有効でもアクセスできない。

```typescript
// 署名検証だけでなく…
const valid = await hmacVerify(env.SIGNING_KEY, data, sig);
if (!valid) return new Response('Invalid signature', { status: 403 });

// KV でリアルタイムの購読状態もチェック
const status = await env.SUBSCRIBERS.get(userToken);
if (status !== 'active')
  return new Response('Subscription inactive', { status: 403 });
```

KV の読み取りレイテンシは P50 で数 ms なので、音声再生に影響はない。

### 2. Firestore と KV の使い分け

|          | Firestore          | Cloudflare KV           |
| -------- | ------------------ | ----------------------- |
| 用途     | ユーザー情報の正本 | エッジでの認可判定      |
| 書き込み | Stripe Webhook 時  | 同上（Functions 経由）  |
| 読み取り | Web クライアント   | Workers（毎リクエスト） |
| 整合性   | 強整合             | 結果整合（数秒ラグ）    |

KV は「キャッシュ」ではなく「認可のための射影」。Firestore が正、KV は Functions が同期する。

### 3. Range ヘッダーの透過的プロキシ

Podcast アプリは音声の部分ダウンロードやシークに `Range` ヘッダーを使う。Workers はこれを Art19 にそのまま転送し、`206 Partial Content` + `Content-Range` もそのまま返す。

```typescript
const rangeHeader = request.headers.get('Range');
if (rangeHeader) {
  headers.set('Range', rangeHeader);
}
// ...
return new Response(upstream.body, {
  status: upstream.status, // 200 or 206
  headers: responseHeaders,
});
```

`upstream.body`（ReadableStream）をそのまま返すことで、Workers のメモリには音声データが載らない。

### 4. Firebase → Cloudflare KV のクロスクラウド同期

Firebase Functions から Cloudflare KV への書き込みは REST API で行う。SDK ではなく直接 `fetch` する。

```typescript
async function kvPut(key: string, value: string): Promise<void> {
  const url = `https://api.cloudflare.com/client/v4/accounts/${accountId}/storage/kv/namespaces/${namespaceId}/values/${key}`;
  await fetch(url, {
    method: 'PUT',
    headers: { Authorization: `Bearer ${apiToken}` },
    body: value,
  });
}
```

Firebase と Cloudflare という 2 つのクラウドにまたがるが、同期タイミングは Stripe Webhook のみ（頻度は低い）なので問題にならない。

---

## 思考の変遷まとめ

```
「フィード2つ取ってマージすれば簡単じゃん」
    ↓
「Alternate Feed の embed が 404 で動かない…」
    ↓
「HTML5 <audio> で直接再生しよう」
    ↓
「待って、Feed URL がクライアントの JS に丸見えだ」
    ↓
「Cloud Functions 経由にして URL を隠そう」
    ↓
「audioUrl を返してもユーザーが共有したら意味ないじゃん…」
    ↓
「Supercast とか使えば？→ 手数料高い。自前で作ろう」
    ↓
「Art19 の Access Token？→ 共有トークンだから漏れたら全滅」
    ↓
「Art19 の JWT？→ <audio> タグで Authorization ヘッダー送れない」
    ↓
「結局プロキシが必要 → それなら Cloudflare Workers で全部やろう」
    ↓
「R2 に音声コピー？→ 同期が面倒。直接プロキシで十分」
    ↓
「Workers (署名付き URL + KV 認可) + Firebase (認証 + 課金) の二段構成」
```

振り返ると、各段階で「これで完璧」と思っていたものが、次の問題を発見するたびに覆されていく過程だった。

---

## やりたかったけどできなかった（まだやっていない）こと

### Art19 の embed プレイヤーをプレミアムでも使いたかった

Art19 の embed プレイヤー（iframe）は再生速度変更やチャプターなど多機能。しかし Alternate Feed のエピソードは embed URL が 404 を返すため使えなかった。HTML5 `<audio>` はシンプルだが機能が少ない。

### Podcast アプリへの自動登録

Supercast は「メール送信 → ワンタップでアプリに追加」のフローがある。自前だと `/feed/:userToken` の URL をユーザーに手動で Podcast アプリに登録してもらう必要がある。UX は劣る。`podcast://` や `overcast://` スキームの活用は今後検討。

### 分析・統計

Art19 には配信分析機能があるが、Workers 経由だとリスナー数などの統計が Art19 側に正しく記録されない可能性がある。Workers 側で独自にログを取る仕組みが将来必要。

### CDN キャッシュ

現状、Workers は毎回 Art19 に音声をフェッチしている。同じエピソードの同じ Range への連続アクセスは Cache API でキャッシュできるはず。ただし、署名付き URL のクエリパラメータが毎回異なるため、キャッシュキーの設計が必要。

---

## 技術的制約とその影響

| 制約                                         | 影響                                 | 対応                                        |
| -------------------------------------------- | ------------------------------------ | ------------------------------------------- |
| Art19 Alternate Feed の embed が 404         | iframe プレイヤーが使えない          | HTML5 `<audio>` で代替                      |
| `<audio>` タグは HTTP ヘッダーを送れない     | Art19 JWT 認証が使えない             | Workers によるプロキシ                      |
| Art19 Access Token は全ユーザー共通          | 1人漏洩 = 全滅                       | 使わない判断                                |
| Cloudflare KV は結果整合                     | 解約後に数秒のラグ                   | 実用上問題なし（数秒レベル）                |
| Vite の `VITE_` 環境変数はバンドルに含まれる | シークレットをクライアントに置けない | サーバーサイド（Functions / Workers）に配置 |

---

## 最終的な技術スタック

| レイヤー              | 技術                 | 役割                             |
| --------------------- | -------------------- | -------------------------------- |
| フロントエンド        | Riot.js + Vite       | SPA、エピソード表示              |
| 認証                  | Firebase Auth        | マジックリンクログイン           |
| ユーザー DB           | Firestore            | 購読状態・premiumFeedToken 保存  |
| 課金                  | Stripe Payment Links | サブスクリプション管理           |
| Webhook 処理          | Firebase Functions   | Stripe → Firestore + KV 同期     |
| フィード配信          | Cloudflare Workers   | RSS 書き換え + 署名付き URL 生成 |
| 音声配信              | Cloudflare Workers   | Art19 からの Range 対応プロキシ  |
| 認可ストア            | Cloudflare KV        | エッジでの購読状態チェック       |
| ホスティング（Art19） | Art19                | 音声ファイルのオリジン           |

---

## 感想

最初は「RSS フィードを2つマージするだけ」だと思っていた。それが、セキュリティを考え始めた途端に芋づる式に問題が出てきて、最終的には Cloudflare Workers で認証プロキシを自作する羽目になった。

一番の学びは **「`<audio>` タグは HTTP ヘッダーを送れない」** という、言われてみれば当たり前だが見落としがちな制約。これがわかった瞬間に、Art19 の JWT 方式も、ブラウザから直接認証付きアクセスする方式も全て不可能だとわかり、プロキシ一択になった。

もう一つは **Firestore と KV の役割分担**。「正本はどこか」「エッジで何を見るか」を最初から設計していたわけではなく、試行錯誤の結果として自然にこの形になった。結果的に、Stripe Webhook を起点として Firestore と KV の両方に書き込む「イベント駆動の射影パターン」になっている。

Supercast に月額を払えばこの苦労は全て不要だった。でも、自前で作ったことで Cloudflare Workers の実力を体感できたし、音声ストリーミングにおける Range ヘッダーの重要性や、クロスクラウドでの認可設計など、プレミアムポッドキャスト配信という実は複雑なドメインの解像度が格段に上がった。

個人開発だからこそできる、「最適解より学びを取る」判断だったと思う。

---

<!-- TODO:
- draw.io でアーキテクチャ図を作成して画像に差し替え
- Mermaid のシーケンス図は Zenn のプレビューで動作確認
- コード例はもう少し削ってもいいかも
- Art19 サポートとのやり取りのスクショ（許可取れれば）
-->
