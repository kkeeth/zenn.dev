---
title: '個人ポッドキャストにプレミアム配信を自前で作った設計と葛藤，そして全部捨てた話'
emoji: '🎙️'
type: 'tech'
topics: ['cloudflare', 'firebase', 'podcast', 'substack', 'architecture']
published: true
---

## はじめに

ポッドキャスト「[雨宿りとWEBの小噺.fm](https://open.spotify.com/show/4ZqUQtob7eJrz9DQV7lPVd)」を運営しています．前々から有料メンバーシップで限定エピソードを配信したいと思っていたのですが，どういうシステムにするか．どうせならこれを機に CloudFlare を触ってみようと思っていたので自前実装をしてみようと思い，Claude Code を用いて実装を進めていましたが，これが「思った以上に難しい」ことに気づきました．

本記事では，技術選定の過程で何を考え，何を試し，何を捨て，最終的にどういうアーキテクチャに着地したか，思考の変遷も含めて書きます．

## TL;DR

- 配信プラットフォーム [Art19](https://art19.com/) の Alternate Feed（プレミアム用フィード）を使う
- 音声ファイルの URL を直接リスナーに渡すとコンテンツ保護できない
- `Cloudflare Workers` でフィードと音声の両方をプロキシし，署名付き URL + KV でアクセス制御
- 課金は `Stripe`，認証・ユーザー管理は `Firebase`，エッジでの認可は `Cloudflare KV`
- 既存サービス（`Memberful`, `Supercast` 等）を使わず自前構築した理由と，そのトレードオフ
  - **素直に Memberful, Supercast 等を使った方が楽**

**が，上記の設計を実装途中まで進めたあと，結局すべて捨てて **Substack** に一本化．その経緯は Phase 6 で．**

---

## 前提：やりたかったこと

1. Art19 で配信しているポッドキャストに**プレミアム限定エピソード**を追加
2. 公式サイト（[Riot.js](https://riot.js.org/) + [vite](https://ja.vite.dev/) 製の SPA）で通常エピソードとプレミアムエピソードを**時系列マージ表示**
3. Podcast アプリ（Apple Podcasts, Overcast 等）でもプレミアムフィードを**購読可能に**
4. 解約したらプレミアムフィードを **即時アクセス停止**

## Phase 1：素朴に実装してみる

### 最初のアイデア

Art19 には Primary Feed（通常公開）と Alternate Feed（限定用）があるので，クライアント側で両方取得してマージ，時系列にソートして表示すれば良いのでは？と．

やってみたら実装は簡単だった．`fetchMergedFeeds()` と言うメソッドを生やし，この中で `Promise.allSettled` を使い，片方が失敗しても表示が止まらないように少し工夫した．

:::details 実際の`fetchMergedFeeds()`メソッドのコード

```js
/**
 * Primary Feed と Premium Feed をマージして時系列でソートする。
 * premiumEpisodesPromise が未指定または失敗時は Primary のみで継続。
 */
export async function fetchMergedFeeds(
  primaryUrl: string,
  premiumEpisodesPromise?: Promise<Episode[]>,
): Promise<Episode[]> {
  const primaryPromise = fetchRSSFeed(primaryUrl);

  if (!premiumEpisodesPromise) {
    return primaryPromise;
  }

  const [primary, premium] = await Promise.allSettled([
    primaryPromise,
    premiumEpisodesPromise,
  ]);

  const primaryEpisodes = primary.status === 'fulfilled' ? primary.value : [];
  const premiumEpisodes = premium.status === 'fulfilled' ? premium.value : [];

  if (premium.status === 'rejected') {
    console.error('[fetchMergedFeeds] premium feed error:', premium.reason);
  }

  // Premium フィードには <link> がないため、同じタイトルのエピソードが
  // 両方のフィードに存在する場合、primary の link を保持しつつ
  // premium の audioUrl と isPremium フラグをマージして重複を除去する
  const premiumByTitle = new Map<string, Episode>();
  for (const ep of premiumEpisodes) {
    premiumByTitle.set(ep.title, ep);
  }

  const mergedPrimary = primaryEpisodes.map((ep) => {
    const premiumEp = premiumByTitle.get(ep.title);
    if (premiumEp) {
      premiumByTitle.delete(ep.title);
      return {
        ...ep,
        isPremium: true,
        audioUrl: premiumEp.audioUrl || ep.audioUrl,
      };
    }
    return ep;
  });

  const merged = [...mergedPrimary, ...premiumByTitle.values()].sort(
    (a, b) => b.pubDateObj.getTime() - a.pubDateObj.getTime(),
  );
  return merged;
}
```

:::

が，すぐに壁に気付く．

### 壁 1：Art19 の Embed Player が Alternate Feed で 404

Art19 の通常エピソードは `https://art19.com/shows/.../episodes/{id}/embed` で iframe プレイヤーが使えるが， **Alternate Feed のエピソードは同じ URL にアクセスしても 404 が返る**．となると，プレミアムエピソードは HTML5 の `<audio>` タグで直接再生するしかないので，RSS の `<enclosure>` から音声 URL を取得して使うことにした．

### 壁 2：Alternate Feed の URL がクライアント JS に丸見え

Vite でビルドすると `VITE_` プレフィックスの環境変数はバンドルに含まれる．つまり **Alternate Feed の URL が誰でも見れてしまう** ので，Feed URL を知っていれば，全エピソードの音声 URL も RSS から取得可能となる．

**これではプレミアムの意味がない．**

## Phase 2：サーバーサイドに逃がす

フィード取得をクライアントからサーバーサイド（`Firebase Cloud Functions` を利用）に移した．

```mermaid
flowchart TD
    A[ブラウザ SPA] -->|GET /getPremiumEpisodes| B[Cloud Function]
    B --> C{Firebase Auth\nuid 検証}
    C -->|認証失敗| D[401 Unauthorized]
    C -->|認証成功| E[(Firestore\nusers/uid.plan)]
    E --> F{plan === 'premium'?}
    F -->|Yes| G[Art19 Alternate Feed 取得]
    G --> H[エピソード一覧\naudioUrl あり]
    F -->|No| I[エピソード一覧\naudioUrl 空文字]
    H --> J[ブラウザ SPA]
    I --> J
```

**解決したこと：** Feed URL がクライアントから消えた（`grep alternate_feeds` でバンドルを確認）．

**解決していないこと：** 返された `audioUrl`（Art19 の直 URL）をプレミアムユーザーがコピーして共有すれば，誰でもアクセスできてしまう．

ということでまだ課題あり．

## Phase 3：他のポッドキャストはどうしている？

ここで一度立ち止まって，他のサービスやポッドキャストの方式を調査した．

### サービス比較

| サービス           | 月額手数料   | 決済手数料                  | 方式                 | 備考                         |
| ------------------ | ------------ | --------------------------- | -------------------- | ---------------------------- |
| **Supercast**      | $0           | 取引の 5.5% + Stripe 手数料 | ユーザーごと固有 RSS | 一番スムーズだが手数料が高い |
| **Memberful**      | $25/月〜     | Stripe 手数料のみ           | ユーザーごと固有 RSS | 月額固定コストが痛い         |
| **Patreon**        | $0           | 5〜12%                      | 独自プラットフォーム | ポッドキャスト特化ではない   |
| **Apple Podcasts** | $19.99/年    | Apple 15〜30%               | Apple 独自           | Apple ユーザー限定           |
| **Spotify**        | $0           | 未公開                      | Spotify 独自         | Spotify ユーザー限定         |
| **rooom**          | $0           | 売上の 8% + 決済手数料      | 独自プラットフォーム | 日本のポッドキャスト特化     |
| **自前構築**       | インフラ実費 | Stripe 3.6%                 | 完全カスタム         | 開発・メンテコスト           |

### 他番組の方式

- **Rebuild.fm**：Web ポータルからダウンロード（RSS ではなくブラウザ経由）
- **backspace.fm（BSM）**： [Ghost](https://ghost.org/) 製．プラットフォームが管理するプレミアムフィード

### 判断

理想は Memberful 等を使って面倒なものやセキュリティを押し付けたいが，個人ポッドキャストで月額 $25 や 5.5% の手数料は厳しい．契約の関係上 Art19 を使い続ける制約もある．ということで，改めて **自前で Supercast 相当のものを作る** 方向に舵を切った．

## Phase 4：Art19 の認証機能を検討 → 断念

色々調べると，Art19 側にもフィードの保護機能がある．実際サポートに問い合わせた（英語メール）ところ，2つの方式を提案された．

### `Access Token` 方式

フィードと音声 URL にクエリパラメータとして共有トークンを付与：

```
https://rss.art19.com/alternate_feeds/...?token=SHARED_SECRET
https://rss.art19.com/episodes/{id}.mp3?token=SHARED_SECRET
```

**問題：** トークンは全ユーザー共通．1人が URL を共有すれば全エピソードにアクセスできる問題が解決しない．ローテーションは可能だが，既存の Podcast アプリに登録済みの URL が無効になってしまう．

### `JWT` 方式

Art19 に事前に公開鍵を登録し，リクエスト時に JWT を `Authorization` ヘッダーで送る．

```
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

**致命的な問題：** `<audio>` タグや Podcast アプリは HTTP リクエストに任意のヘッダーを付与できない．つまり **JWT を使うにはどのみちプロキシが必要** になる．プロキシを立てるなら JWT の意味が薄れる（プロキシ ↔ Art19 間はサーバー間通信なので，他の方法でも保護できる）．

```
❌ <audio src="https://art19.com/..."> → Authorization ヘッダーを付けられない
✅ <audio src="https://my-proxy/..."> → プロキシが Art19 に Authorization 付きで中継
    → でもそれならプロキシ側で認証すれば JWT 不要
```

### 結論

どちらの方式も「プロキシなしでクライアント / Podcast アプリから直接アクセス」を実現できない．**プロキシを建てるなら，Art19 の認証機能に依存せず自前で制御した方がシンプル**．

## Phase 5：自前プロキシ設計

じゃあどうするか，色々 Claude Code 君と検討していたが，[Cloudflare Workers](https://www.cloudflare.com/ja-jp/developer-platform/products/workers/) でいくことにした．

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

以上を Workers が全て満たしていた．

### R2 を使わなかった理由

当初は Art19 の音声を Cloudflare R2 にコピーしてから配信することも検討した．

```
（検討案）Art19 → R2 にコピー → R2 から署名付き URL で配信
（採用案）Art19 → Workers が直接プロキシ → 署名付き URL で配信
```

**R2 を選ばなかった理由：**

- Art19 に新エピソードが追加されたとき，R2 への同期が必要（cron? webhook?）
- Art19 側でエピソードが更新・削除されたときの同期問題
- R2 のストレージコスト（音声ファイルは大きい）
- **プロキシで十分**：Workers は Art19 からストリーミングでそのままクライアントに転送するので，メモリにも載らない

### 全体像

ということで，まとめるとこうなった．

```mermaid
graph LR
  subgraph Stripe["💳 Stripe"]
    S_PL["Payment Link"]
    S_WH["⚡ Webhook"]
    S_CP["Customer Portal"]
  end

  subgraph Cloudflare["☁️ Cloudflare"]
    CF_KV[("🗄️ KV\nSUBSCRIBERS")]
    CF_WF{{"⬡ Worker\n/feed/:userToken"}}
    CF_WA{{"⬡ Worker\n/audio/:episodeId"}}
  end

  subgraph Firebase["🔥 Firebase"]
    FB_FN["⚙️ Cloud Functions\n(stripeWebhook)"]
    FB_AUTH["🔒 Auth\n(Magic Link)"]
    FB_FS[("📁 Firestore\nusers/{uid}")]
  end


  subgraph Art19["🎙️ Art19"]
    A19_RSS["📄 Alternate Feed\n(Premium RSS)"]
    A19_MP3["🎵 Audio\n(.mp3)"]
  end

  USER(("👤 User"))
  APP["📱 Podcast App"]
  WEB["💻 Web Client\n(Riot.js SPA)"]

  S_WH -->|"checkout / cancel"| FB_FN
  FB_FN -->|write| FB_FS
  FB_FN -.->|"REST API PUT/DELETE"| CF_KV

  CF_WF -->|check| CF_KV
  CF_WA -->|check| CF_KV

  CF_WF -->|fetch RSS| A19_RSS
  CF_WA -->|"Range proxy (206)"| A19_MP3

  USER -->|決済| S_PL
  USER -->|解約| S_CP
  USER --> WEB
  WEB -->|login| FB_AUTH
  WEB -->|get token| FB_FS
  WEB -.->|"/feed/:token"| CF_WF
  APP -->|"/feed/:userToken"| CF_WF
  APP -->|"/audio/:id?sig=..."| CF_WA
```

## Phase 6：設計を全部捨てて `Substack` に一本化

### 壁にぶつかったのは技術ではなく「メンテナンスコストと維持コスト，セキュリティ」

Phase 5 の設計を実装し始めたとき，改めて月額ランニングコストを試算した．

| 項目                                | 月額概算                 |
| ----------------------------------- | ------------------------ |
| Cloudflare Workers Paid（超過分）   | $5〜                     |
| Cloudflare KV（書き込み・読み取り） | $0.5〜                   |
| Firebase Functions（Blaze プラン）  | $0〜（無料枠内の見込み） |
| Firestore（リード数次第）           | $0〜数百円               |
| Stripe 手数料                       | 売上の 3.6%              |

金額だけなら許容範囲だった．問題は **メンテナンスコスト**．

- Cloudflare Workers × Firebase Functions の **クロスクラウド同期** のバグリスク
- HMAC 署名の鍵ローテーション，KV の TTL 管理
- Stripe Webhook の冪等性保証，失敗時リトライ
- セキュリティ監査（誰が触っても把握できる人間が自分一人）

「個人ポッドキャストのプレミアム配信のために，これを全部自分で運用するのか」と冷静に考えたとき，**本来やりたいことは配信であって，インフラの運用ではない** という当たり前の事実に行き着いた．また，昨今の AI の進化によるサプライチェーン攻撃の恐怖が凄まじく，自分でリスクを背負うのはやはり止めたほうが良いと判断．

ということで，外部のサービスの比較検討にシフト．

### Substack に決定

月額コストを下げる選択肢を探していたとき，Substack が真っ先に浮かんだ．自分も [別のポッドキャスト番組](https://kkeeth.substack.com/podcast) の配信を Substack からしていたからだ．

:::details ちなみに
私が Substack のアカウントを作ったのは 2024年10月で，昨今のとあるインフルエンサーが使い始めた事による流行以前から利用している．何となくこのウェーブに乗っかったと思われるのが癪だったので一応．
::::

| 項目               | Substack                        |
| ------------------ | ------------------------------- |
| 手数料             | 売上の 10%（Stripe 手数料込み） |
| 月額固定費         | $0                              |
| 音声ホスティング   | あり                            |
| メンバーシップ管理 | あり                            |
| RSS フィード       | あり（公開）                    |
| Podcast アプリ対応 | あり                            |

手数料 10% は Supercast の 5.5% より高い．ただし **月額固定費がゼロ** なので，配信が軌道に乗るまでは圧倒的に低コスト．

さらに，決済と会員登録・ログイン認証を全て Substack に押し付けられるので，自前で持つ攻撃対象領域をほぼゼロにした．

:::message

- オウンドメディアで利用しているライブラリへのサプライチェーン攻撃
- Substack の RSS に悪意あるコンテンツが含まれた場合の XSS（可能性は極めて低い）

などは残っている
:::

### RSS フィードの「仕様」が全てを解決した

Substack の公開 RSS フィードを確認したところ，

```xml
<!-- 無料エピソード：<enclosure> あり（音声 URL が公開される） -->
<item>
  <title>無料エピソードタイトル</title>
  <enclosure url="https://..." type="audio/mpeg" length="..." />
</item>

<!-- 有料エピソード：<enclosure> なし（音声 URL は含まれない） -->
<item>
  <title>【会員限定】有料エピソードタイトル</title>
  <!-- enclosure タグがない -->
</item>
```

**`<enclosure>` の有無だけでプレミアム判定できる．** これはつまり：

```typescript
const isPremium = !audioUrl; // enclosure なし = locked
```

認証も署名も KV も不要．RSS を取得して `audioUrl` が空かどうかを見るだけ．

### 最終アーキテクチャ（シンプル版）

```
ART19 RSS（公開）──────────────────┐
                                   ├─ Promise.allSettled でマージ → SPA で表示
Substack RSS（公開）───────────────┘

無料エピソード（ART19）   → HTML5 <audio> で再生
有料エピソード（Substack）→ 鍵アイコン + Substack へのリンク　※一部はサンプルとして無料公開予定
                            （認証・再生は Substack 側に委譲）
```

自サイトは **「RSS をマージして表示するだけ」** になった．Workers も Firebase も Stripe も Cloudflare KV も全て不要になったため，Phase 5 で作りかけていたコードはすべて捨てた．

### 捨てたもの

| 捨てたもの                         | 理由                                        |
| ---------------------------------- | ------------------------------------------- |
| Cloudflare Workers（音声プロキシ） | Substack が音声ホスティングと認証を担う     |
| Cloudflare KV                      | 購読状態の管理が Substack に移った          |
| Firebase Auth                      | ログインが不要になった（Substack 側で管理） |
| Firestore（premiumFeedToken）      | 同上                                        |
| Stripe Webhook 処理                | 課金が Substack に移った                    |
| HMAC 署名付き URL                  | 音声 URL 保護が不要になった                 |

### トレードオフ

自前構築を捨てたことで失ったものもある．

| 失ったこと                                         | 影響                            |
| -------------------------------------------------- | ------------------------------- |
| 手数料が 3.6% → 10%                                | 売上規模が大きくなったら再検討  |
| 再生 UI のカスタマイズ不可                         | Substack の再生 UI に依存       |
| アクセス解析の完全な把握                           | Substack のダッシュボードに依存 |
| Podcast アプリで会員限定フィードを購読させるフロー | Substack の仕組みに従う         |

**ただし今の段階では，これらは全てトレードオフとして許容できる．**

月額固定費ゼロ・開発工数ゼロ・メンテナンスコストゼロ（自前インフラとして）と比較すれば，10% の手数料は「配信が軌道に乗るまでの保険料」と考えられる．

## 思考の変遷まとめ

```
「フィード2つ取ってマージすれば簡単じゃん」
    ↓
「Alternate Feed の embed が 404 で動かない…」
    ↓
「HTML5 <audio> で直接再生しよう」
    ↓
「待って，Feed URL がクライアントの JS に丸見えだ」
    ↓
「Cloud Functions 経由にして URL を隠そう」
    ↓
「audioUrl を返してもユーザーが共有したら意味ないじゃん…」
    ↓
「Supercast とか使えば？→ 手数料高い．自前で作ろう」
    ↓
「Art19 の Access Token？→ 共有トークンだから漏れたら全滅」
    ↓
「Art19 の JWT？→ <audio> タグで Authorization ヘッダー送れない」
    ↓
「結局プロキシが必要 → それなら Cloudflare Workers で全部やろう」
    ↓
「R2 に音声コピー？→ 同期が面倒．直接プロキシで十分」
    ↓
「Workers (署名付き URL + KV 認可) + Firebase (認証 + 課金) の二段構成」
    ↓
「ランニングコスト・メンテコストを改めて試算…個人運営でこれを自分で維持するのか？」
    ↓
「Substack がポッドキャスト配信に対応しているのを知る」
    ↓
「Substack の RSS を確認したら有料エピソードは <enclosure> がない」
    ↓
「isPremium = !audioUrl だけで判定できる → 認証もプロキシも不要」
    ↓
「認証・課金・再生を全部 Substack に委譲．自サイトは RSS をマージして表示するだけ」
    ↓
「作りかけていた Workers / Firebase / Stripe / KV を全部削除」
```

振り返ると，各段階で「これでいける！」と思っていたものが，次の問題を発見するたびに覆されていく過程だった．

## 技術的制約とその影響

これも改めて表にまとめてみました．参考に．

| 制約                                         | 影響                                 | 対応                                        |
| -------------------------------------------- | ------------------------------------ | ------------------------------------------- |
| Art19 Alternate Feed の embed が 404         | iframe プレイヤーが使えない          | HTML5 `<audio>` で代替                      |
| `<audio>` タグは HTTP ヘッダーを送れない     | Art19 JWT 認証が使えない             | Workers によるプロキシ                      |
| Art19 Access Token は全ユーザー共通          | 1人漏洩 = 全滅                       | 使わない判断                                |
| Cloudflare KV は結果整合                     | 解約後に数秒のラグ                   | 実用上問題なし（数秒レベル）                |
| Vite の `VITE_` 環境変数はバンドルに含まれる | シークレットをクライアントに置けない | サーバーサイド（Functions / Workers）に配置 |

---

## 最終的な技術スタック

### 当初設計（Phase 5まで・没）

| レイヤー       | 技術                 | 役割                             |
| -------------- | -------------------- | -------------------------------- |
| フロントエンド | Riot.js + Vite       | SPA，エピソード表示              |
| 認証           | Firebase Auth        | マジックリンクログイン           |
| ユーザー DB    | Firestore            | 購読状態・premiumFeedToken 保存  |
| 課金           | Stripe Payment Links | サブスクリプション管理           |
| Webhook 処理   | Firebase Functions   | Stripe → Firestore + KV 同期     |
| フィード配信   | Cloudflare Workers   | RSS 書き換え + 署名付き URL 生成 |
| 音声配信       | Cloudflare Workers   | Art19 からの Range 対応プロキシ  |
| 認可ストア     | Cloudflare KV        | エッジでの購読状態チェック       |
| ホスティング   | Art19                | 音声ファイルのオリジン           |

### 実際に採用したスタック（Phase 6）

| レイヤー       | 技術                                    | 役割                                                      |
| -------------- | --------------------------------------- | --------------------------------------------------------- |
| フロントエンド | Riot.js + Vite                          | SPA，エピソード表示                                       |
| 公開フィード   | ART19                                   | 通常エピソードの RSS + 音声ホスティング                   |
| 会員フィード   | Substack                                | 会員限定エピソードの RSS + 認証 + 課金 + 音声ホスティング |
| CORS プロキシ  | Cloudflare Workers（1ファイル・約45行） | Substack RSS の CORS 回避のみ                             |
| ホスティング   | Firebase Hosting                        | SPA の配信                                                |

Workers は CORS プロキシに特化したシンプルな実装になった．

---

## 感想

### Phase 5 まで

最初は「RSS フィードを2つマージするだけ」だと思っていた．それが，セキュリティを考え始めた途端に芋づる式に問題が出てきて，最終的には Cloudflare Workers で認証プロキシを自作する羽目になった．

一番の学びは **「`<audio>` タグは HTTP ヘッダーを送れない」** という，言われてみれば当たり前だが見落としがちな制約．これがわかった瞬間に，Art19 の JWT 方式も，ブラウザから直接認証付きアクセスする方式も全て不可能だとわかり，プロキシ一択になった．

もう一つは **Firestore と KV の役割分担**．「音源はどこか」「エッジで何を見るか」を最初から設計していたわけではなく，試行錯誤の結果として自然にこの形になった．結果的に，Stripe Webhook を起点として Firestore と KV の両方に書き込む「イベント駆動の射影パターン」になっている．

Supercast 等に月額を払えばこの苦労は全て不要だった．でも，自前で作ったことで Cloudflare Workers の実力を体感できたし，音声ストリーミングにおける Range ヘッダーの重要性や，クロスクラウドでの認可設計など，プレミアムポッドキャスト配信という実は複雑なドメインの解像度が格段に上がった．

### 設計を捨てる判断について

Phase 5 まで「自前で Supercast を作る」方向で進めていたが，コスト・セキュリティの観点で捨てることにした．まぁこれが今のところはこれが無難だろうなと満足している．これから自前実装をしようと考えている方の参考になれば幸い．ちなみに「ここまで設計したのに捨てるのは勿体ない」というサンクコスト的な感情も未だにある笑

ただ冷静に見ると，**自分がやりたいのは配信であってインフラの運用ではない** という事実は変わらない．Phase 1〜5 で積み上げた設計と実装は，「何がなぜ問題になるか」を深く理解するための過程として意味があった．問題の解像度が上がったことで，「もっとシンプルな解答」を受け入れられるようになっただけだ．個人開発だからこそできる，「最適解より学びを取る」判断だったと思う．

ではでは．
