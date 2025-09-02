---
title: 'S3でギャラリーフロントエンドを構築'
---

# S3でギャラリーフロントエンドを構築 - ArtVaultの基盤作成

この章では，S3を使ってアートギャラリー「ArtVault」のフロントエンドを構築します．S3の静的Webサイトホスティング機能を活用して，デジタルアート作品を美しく表示するギャラリーページを作成します．

## 学習目標

- S3の静的Webサイトホスティング機能の理解
- バケットポリシーでのパブリックアクセス制御
- HTMLとJavaScriptによるギャラリーUIの構築
- S3 APIを使ったオブジェクト一覧取得

## 1. フロントエンド用バケットの作成

まず，Webサイトホスティング用の専用S3バケットを作成します．

`scripts/setup-frontend.mjs` を作成：

```js
import { S3Client, CreateBucketCommand, PutBucketWebsiteCommand, PutBucketPolicyCommand } from '@aws-sdk/client-s3';

const config = {
  endpoint: 'http://localhost:4566',
  region: 'ap-northeast-1',
  credentials: {
    accessKeyId: 'test',
    secretAccessKey: 'test'
  }
};

const s3Client = new S3Client({ ...config, forcePathStyle: true });

async function setupFrontendBucket() {
  const bucketName = 'artvault-frontend';

  try {
    console.log('🎨 フロントエンド用バケットをセットアップ中...');

    // バケット作成
    await s3Client.send(new CreateBucketCommand({
      Bucket: bucketName,
      CreateBucketConfiguration: { LocationConstraint: config.region }
    }));
    console.log(`✅ S3 バケット "${bucketName}" を作成しました`);

    // 静的Webサイトホスティング設定
    await s3Client.send(new PutBucketWebsiteCommand({
      Bucket: bucketName,
      WebsiteConfiguration: {
        IndexDocument: { Suffix: 'index.html' },
        ErrorDocument: { Key: 'error.html' }
      }
    }));
    console.log('✅ 静的Webサイトホスティングを有効化しました');

    // パブリックリード用のバケットポリシー
    const policy = {
      Version: '2012-10-17',
      Statement: [
        {
          Sid: 'PublicReadGetObject',
          Effect: 'Allow',
          Principal: '*',
          Action: 's3:GetObject',
          Resource: `arn:aws:s3:::${bucketName}/*`
        }
      ]
    };

    await s3Client.send(new PutBucketPolicyCommand({
      Bucket: bucketName,
      Policy: JSON.stringify(policy)
    }));
    console.log('✅ パブリックアクセス用バケットポリシーを設定しました');

    console.log('🚀 フロントエンドセットアップ完了！');
    console.log(`🌐 サイトURL: http://${bucketName}.s3-website.localhost:4566/`);

  } catch (error) {
    if (error.$metadata?.httpStatusCode === 409 || error.name === 'BucketAlreadyExists') {
      console.log(`ℹ️  バケット "${bucketName}" は既に存在します`);
    } else {
      console.error('❌ セットアップエラー:', error.message);
    }
  }
}

setupFrontendBucket();
```

`package.json` にスクリプトを追加：

```diff
  {
    "scripts": {
-     "setup:resources": "node scripts/setup-resources.mjs"
+     "setup:resources": "node scripts/setup-resources.mjs",
+     "setup:frontend": "node scripts/setup-frontend.mjs"
    }
  }
```

## 2. ギャラリーのHTMLテンプレート作成

`frontend/index.html` を作成：

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ArtVault - デジタルアートギャラリー</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: 'Arial', sans-serif;
            background: linear-gradient(135deg, #1a1a1a 0%, #2d2d2d 100%);
            color: #ffffff;
            min-height: 100vh;
        }

        .header {
            text-align: center;
            padding: 40px 20px;
            border-bottom: 1px solid #404040;
        }

        .header h1 {
            font-size: 3rem;
            margin-bottom: 10px;
            background: linear-gradient(45deg, #ff6b6b, #4ecdc4);
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
            background-clip: text;
        }

        .header p {
            color: #cccccc;
            font-size: 1.1rem;
        }

        .upload-section {
            padding: 20px;
            text-align: center;
            border-bottom: 1px solid #404040;
        }

        .upload-btn {
            background: linear-gradient(45deg, #ff6b6b, #4ecdc4);
            color: white;
            border: none;
            padding: 12px 30px;
            border-radius: 25px;
            font-size: 1rem;
            cursor: pointer;
            transition: transform 0.2s;
        }

        .upload-btn:hover {
            transform: translateY(-2px);
        }

        .gallery {
            padding: 40px 20px;
            max-width: 1200px;
            margin: 0 auto;
        }

        .gallery-grid {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
            gap: 20px;
            margin-top: 20px;
        }

        .artwork-card {
            background: #333333;
            border-radius: 15px;
            padding: 15px;
            box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
            transition: transform 0.3s, box-shadow 0.3s;
        }

        .artwork-card:hover {
            transform: translateY(-5px);
            box-shadow: 0 12px 40px rgba(0, 0, 0, 0.4);
        }

        .artwork-image {
            width: 100%;
            height: 200px;
            object-fit: cover;
            border-radius: 10px;
            margin-bottom: 15px;
        }

        .artwork-info h3 {
            margin-bottom: 5px;
            color: #ff6b6b;
        }

        .artwork-info p {
            color: #cccccc;
            font-size: 0.9rem;
            margin-bottom: 5px;
        }

        .loading {
            text-align: center;
            padding: 40px;
            font-size: 1.2rem;
            color: #cccccc;
        }

        .error {
            text-align: center;
            padding: 40px;
            color: #ff6b6b;
            font-size: 1.1rem;
        }

        .no-artworks {
            text-align: center;
            padding: 60px 20px;
            color: #cccccc;
        }

        .no-artworks h2 {
            margin-bottom: 15px;
            color: #666666;
        }
    </style>
</head>
<body>
    <div class="header">
        <h1>ArtVault</h1>
        <p>デジタルアート作品の美しいギャラリー</p>
    </div>

    <div class="upload-section">
        <button class="upload-btn" onclick="uploadArtwork()">
            🎨 新しい作品をアップロード
        </button>
    </div>

    <div class="gallery">
        <div id="loading" class="loading">
            🎨 ギャラリーを読み込み中...
        </div>

        <div id="error" class="error" style="display: none;">
            ❌ ギャラリーの読み込みに失敗しました
        </div>

        <div id="gallery-grid" class="gallery-grid" style="display: none;">
            <!-- アート作品がここに動的に表示されます -->
        </div>

        <div id="no-artworks" class="no-artworks" style="display: none;">
            <h2>📷 まだ作品がありません</h2>
            <p>最初の作品をアップロードして，ギャラリーを始めましょう！</p>
        </div>
    </div>

    <script src="gallery.js"></script>
</body>
</html>
```

## 3. ギャラリーのJavaScriptロジック作成

`frontend/gallery.js` を作成：

```javascript
// LocalStack S3 設定
const S3_ENDPOINT = 'http://localhost:4566';
const GALLERY_BUCKET = 'artvault-gallery';
const REGION = 'ap-northeast-1';

class ArtVaultGallery {
    constructor() {
        this.artworks = [];
        this.initializeGallery();
    }

    async initializeGallery() {
        try {
            await this.loadArtworks();
            this.renderGallery();
        } catch (error) {
            console.error('ギャラリー初期化エラー:', error);
            this.showError();
        }
    }

    async loadArtworks() {
        // LocalStack S3 API を使用してオブジェクト一覧を取得
        const response = await fetch(
            `${S3_ENDPOINT}/${GALLERY_BUCKET}?list-type=2`,
            {
                method: 'GET',
                headers: {
                    'Authorization': 'AWS4-HMAC-SHA256 Credential=test/20231201/ap-northeast-1/s3/aws4_request, SignedHeaders=host;x-amz-date, Signature=test'
                }
            }
        );

        if (!response.ok) {
            throw new Error(`S3 API Error: ${response.status}`);
        }

        const xmlText = await response.text();
        this.parseS3Response(xmlText);
    }

    parseS3Response(xmlText) {
        // 簡易的なXMLパースでオブジェクト一覧を取得
        const parser = new DOMParser();
        const doc = parser.parseFromString(xmlText, 'text/xml');
        const contents = doc.querySelectorAll('Contents');

        this.artworks = Array.from(contents).map(content => {
            const key = content.querySelector('Key')?.textContent;
            const lastModified = content.querySelector('LastModified')?.textContent;
            const size = content.querySelector('Size')?.textContent;

            if (key && this.isImageFile(key)) {
                return {
                    key,
                    name: this.extractFilename(key),
                    url: `${S3_ENDPOINT}/${GALLERY_BUCKET}/${key}`,
                    uploadDate: new Date(lastModified).toLocaleDateString('ja-JP'),
                    size: this.formatFileSize(parseInt(size))
                };
            }
            return null;
        }).filter(Boolean);
    }

    isImageFile(filename) {
        const imageExtensions = ['.jpg', '.jpeg', '.png', '.gif', '.webp', '.svg'];
        return imageExtensions.some(ext =>
            filename.toLowerCase().endsWith(ext)
        );
    }

    extractFilename(key) {
        // パスからファイル名を抽出し，拡張子を除去
        const filename = key.split('/').pop();
        return filename.replace(/\.[^/.]+$/, '');
    }

    formatFileSize(bytes) {
        if (bytes === 0) return '0 Bytes';
        const k = 1024;
        const sizes = ['Bytes', 'KB', 'MB', 'GB'];
        const i = Math.floor(Math.log(bytes) / Math.log(k));
        return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i];
    }

    renderGallery() {
        const loadingEl = document.getElementById('loading');
        const errorEl = document.getElementById('error');
        const galleryEl = document.getElementById('gallery-grid');
        const noArtworksEl = document.getElementById('no-artworks');

        loadingEl.style.display = 'none';
        errorEl.style.display = 'none';

        if (this.artworks.length === 0) {
            noArtworksEl.style.display = 'block';
            galleryEl.style.display = 'none';
            return;
        }

        noArtworksEl.style.display = 'none';
        galleryEl.style.display = 'grid';

        galleryEl.innerHTML = this.artworks.map(artwork => `
            <div class="artwork-card">
                <img src="${artwork.url}"
                     alt="${artwork.name}"
                     class="artwork-image"
                     onerror="this.src='data:image/svg+xml,<svg xmlns=\\"http://www.w3.org/2000/svg\\" width=\\"300\\" height=\\"200\\" viewBox=\\"0 0 300 200\\"><rect width=\\"300\\" height=\\"200\\" fill=\\"#333\\"/><text x=\\"150\\" y=\\"100\\" fill=\\"white\\" text-anchor=\\"middle\\" dominant-baseline=\\"middle\\">画像読み込みエラー</text></svg>'"
                     loading="lazy">
                <div class="artwork-info">
                    <h3>${artwork.name}</h3>
                    <p>📅 アップロード日: ${artwork.uploadDate}</p>
                    <p>📏 ファイルサイズ: ${artwork.size}</p>
                </div>
            </div>
        `).join('');
    }

    showError() {
        const loadingEl = document.getElementById('loading');
        const errorEl = document.getElementById('error');
        const galleryEl = document.getElementById('gallery-grid');
        const noArtworksEl = document.getElementById('no-artworks');

        loadingEl.style.display = 'none';
        errorEl.style.display = 'block';
        galleryEl.style.display = 'none';
        noArtworksEl.style.display = 'none';
    }
}

// アップロード機能（次章で実装予定）
function uploadArtwork() {
    alert('アップロード機能は次章で実装します！\n今は手動でS3バケットに画像ファイルをアップロードしてください．');
}

// ページ読み込み時にギャラリーを初期化
document.addEventListener('DOMContentLoaded', () => {
    new ArtVaultGallery();
});
```

## 4. サンプルアート作品の準備

テスト用のサンプル画像をアップロードするスクリプトを作成します．

`scripts/upload-samples.mjs` を作成：

```js
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { readFileSync } from 'fs';
import { join, dirname } from 'path';
import { fileURLToPath } from 'url';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

const config = {
  endpoint: 'http://localhost:4566',
  region: 'ap-northeast-1',
  credentials: {
    accessKeyId: 'test',
    secretAccessKey: 'test'
  }
};

const s3Client = new S3Client({ ...config, forcePathStyle: true });

// サンプル用のSVG画像を生成
const sampleArtworks = [
  {
    name: 'abstract-waves.svg',
    content: `<svg width="400" height="300" xmlns="http://www.w3.org/2000/svg">
      <defs>
        <linearGradient id="grad1" x1="0%" y1="0%" x2="100%" y2="100%">
          <stop offset="0%" style="stop-color:#ff6b6b;stop-opacity:1" />
          <stop offset="100%" style="stop-color:#4ecdc4;stop-opacity:1" />
        </linearGradient>
      </defs>
      <rect width="400" height="300" fill="#1a1a1a"/>
      <path d="M0,150 Q100,50 200,150 T400,150" stroke="url(#grad1)" stroke-width="8" fill="none"/>
      <path d="M0,200 Q100,100 200,200 T400,200" stroke="url(#grad1)" stroke-width="6" fill="none" opacity="0.7"/>
      <text x="200" y="250" fill="#ffffff" text-anchor="middle" font-family="Arial" font-size="16">Abstract Waves</text>
    </svg>`
  },
  {
    name: 'geometric-pattern.svg',
    content: `<svg width="400" height="300" xmlns="http://www.w3.org/2000/svg">
      <rect width="400" height="300" fill="#2d2d2d"/>
      <polygon points="200,50 350,150 200,250 50,150" fill="#ff6b6b" opacity="0.8"/>
      <polygon points="200,80 320,140 200,200 80,140" fill="#4ecdc4" opacity="0.6"/>
      <circle cx="200" cy="150" r="30" fill="#ffffff" opacity="0.4"/>
      <text x="200" y="280" fill="#ffffff" text-anchor="middle" font-family="Arial" font-size="16">Geometric Pattern</text>
    </svg>`
  },
  {
    name: 'digital-landscape.svg',
    content: `<svg width="400" height="300" xmlns="http://www.w3.org/2000/svg">
      <defs>
        <linearGradient id="sky" x1="0%" y1="0%" x2="0%" y2="100%">
          <stop offset="0%" style="stop-color:#1a1a2e;stop-opacity:1" />
          <stop offset="100%" style="stop-color:#16213e;stop-opacity:1" />
        </linearGradient>
      </defs>
      <rect width="400" height="300" fill="url(#sky)"/>
      <polygon points="0,200 100,120 200,160 300,100 400,140 400,300 0,300" fill="#0f4c75"/>
      <circle cx="350" cy="80" r="25" fill="#ff6b6b"/>
      <text x="200" y="280" fill="#ffffff" text-anchor="middle" font-family="Arial" font-size="16">Digital Landscape</text>
    </svg>`
  }
];

async function uploadSamples() {
  console.log('🎨 サンプルアート作品をアップロード中...');

  for (const artwork of sampleArtworks) {
    try {
      await s3Client.send(new PutObjectCommand({
        Bucket: 'artvault-gallery',
        Key: `artworks/${artwork.name}`,
        Body: artwork.content,
        ContentType: 'image/svg+xml'
      }));
      console.log(`✅ "${artwork.name}" をアップロードしました`);
    } catch (error) {
      console.error(`❌ "${artwork.name}" のアップロードに失敗:`, error.message);
    }
  }

  console.log('🚀 サンプルアップロード完了！');
}

uploadSamples();
```

## 5. フロントエンドのデプロイ

`scripts/deploy-frontend.mjs` を作成：

```js
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { readFileSync } from 'fs';
import { join, dirname } from 'path';
import { fileURLToPath } from 'url';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

const config = {
  endpoint: 'http://localhost:4566',
  region: 'ap-northeast-1',
  credentials: {
    accessKeyId: 'test',
    secretAccessKey: 'test'
  }
};

const s3Client = new S3Client({ ...config, forcePathStyle: true });

async function deployFrontend() {
  console.log('🚀 フロントエンドをデプロイ中...');

  const files = [
    { local: '../frontend/index.html', s3Key: 'index.html', contentType: 'text/html' },
    { local: '../frontend/gallery.js', s3Key: 'gallery.js', contentType: 'application/javascript' }
  ];

  for (const file of files) {
    try {
      const filePath = join(__dirname, file.local);
      const content = readFileSync(filePath, 'utf8');

      await s3Client.send(new PutObjectCommand({
        Bucket: 'artvault-frontend',
        Key: file.s3Key,
        Body: content,
        ContentType: file.contentType
      }));

      console.log(`✅ ${file.s3Key} をデプロイしました`);
    } catch (error) {
      console.error(`❌ ${file.s3Key} のデプロイに失敗:`, error.message);
    }
  }

  console.log('🌐 デプロイ完了！');
  console.log('サイトURL: http://artvault-frontend.s3-website.localhost:4566/');
}

deployFrontend();
```

## 6. package.json の更新

新しいスクリプトを `package.json` に追加：

```json
{
  "scripts": {
    "setup:frontend": "node scripts/setup-frontend.mjs",
    "upload:samples": "node scripts/upload-samples.mjs",
    "deploy:frontend": "node scripts/deploy-frontend.mjs",
    "dev:gallery": "concurrently \"pnpm start:localstack\" \"pnpm upload:samples\" \"pnpm deploy:frontend\""
  }
}
```

## 7. 動作確認

ギャラリーが正しく動作することを確認します：

```bash
# LocalStack起動（まだ起動していない場合）
$ pnpm start:localstack

# 基本リソースセットアップ
$ pnpm setup:resources

# フロントエンド用バケットセットアップ
$ pnpm setup:frontend

# サンプル作品アップロード
$ pnpm upload:samples

# フロントエンドデプロイ
$ pnpm deploy:frontend
```

ブラウザで http://artvault-frontend.s3-website.localhost:4566/ にアクセスして，サンプルアート作品が表示されることを確認してください．

## 8. プロジェクト構成

この章で作成されるファイル構成：

```
artvault-gallery/
├── frontend/
│   ├── index.html          # メインHTMLファイル
│   └── gallery.js          # ギャラリーのJavaScriptロジック
├── scripts/
│   ├── setup-resources.mjs     # 基本リソースセットアップ
│   ├── setup-frontend.mjs     # フロントエンド用バケット設定
│   ├── upload-samples.mjs     # サンプル作品アップロード
│   └── deploy-frontend.mjs    # フロントエンドデプロイ
├── docker-compose.yml
├── package.json
└── README.md
```

## トラブルシューティング

### ギャラリーに画像が表示されない場合
- S3バケットに作品がアップロードされているか確認: `pnpm aws:local s3 ls s3://artvault-gallery --recursive`
- バケットポリシーが正しく設定されているか確認
- ブラウザの開発者ツールでネットワークエラーを確認

### S3 APIエラーが発生する場合
- LocalStackが正常に起動しているか確認: `curl http://localhost:4566/_localstack/health`
- CORS設定が必要な場合があります（次章で詳しく説明）

### 静的ウェブサイトにアクセスできない場合
- バケットのWebサイト設定を確認: `pnpm aws:local s3api get-bucket-website --bucket artvault-frontend`

## 次の章へ

S3を使った基本的なギャラリーが完成しました！次章では，DynamoDBを使って作品のメタデータ（タイトル，説明，タグなど）を管理し，より本格的なギャラリー機能を実装していきます．作品の検索機能や詳細情報表示などの機能を追加していきましょう．