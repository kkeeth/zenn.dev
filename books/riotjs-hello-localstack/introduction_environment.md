---
title: '環境構築'
---

# 環境構築 - ArtVault ギャラリー開発の準備

この章では、デジタルアート作品ギャラリー「ArtVault」を LocalStack を使って構築するための環境を準備します。

## 必要な前提知識

- JavaScript/Node.js の基本知識
- AWS の基本概念（S3、Lambda、DynamoDB など）への理解
- ターミナル操作の基本

## 開発環境の要件

- Node.js 18 以上
- Docker Desktop
- AWS CLI
- お好みのテキストエディタ（VS Code 推奨）

## LocalStack とは

LocalStack は AWS クラウドサービスをローカル環境で再現するツールです。実際の AWS を使わずに開発・テストができるため、以下のメリットがあります：

- **コスト削減**: AWS の課金を気にせず開発可能
- **高速開発**: ネットワーク遅延なしで即座にテスト
- **安全性**: 本番環境に影響を与えずに実験可能

## 1. プロジェクト初期化

まず、プロジェクトディレクトリを作成し、必要なパッケージをインストールします。

```bash
$ mkdir artvault-gallery
$ cd artvault-gallery
$ pnpm init

# git 管理したい方はここで初期化
$ git init
```

必要な依存関係をインストール：

```bash
# LocalStack と AWS SDK
$ pnpm install --dev localstack @aws-sdk/client-s3 @aws-sdk/client-dynamodb @aws-sdk/client-lambda

# 開発用ツール
$ pnpm install --dev concurrently
```

## 2. LocalStack のセットアップ

### Docker Compose の設定

`docker-compose.yml` を作成：

```yaml
version: '3.8'
services:
  localstack:
    container_name: artvault-localstack
    image: localstack/localstack:latest
    ports:
      - "4566:4566"
    environment:
      - DEBUG=1
      - SERVICES=s3,dynamodb,lambda,apigateway,sns,sqs,iam,cloudformation
      - DATA_DIR=/var/lib/localstack
      - PERSISTENCE=1
    volumes:
      - localstack-data:/var/lib/localstack
      - /var/run/docker.sock:/var/run/docker.sock

volumes:
  localstack-data:
```

### LocalStack の起動

```bash
$ docker-compose up -d
```

起動確認：

```bash
$ curl http://localhost:4566/_localstack/health
```

正常に起動していれば、各サービスの状態が表示されます。

## 3. AWS CLI の設定

LocalStack 用の AWS CLI プロファイルを設定：

```bash
$ aws configure set aws_access_key_id test --profile localstack
$ aws configure set aws_secret_access_key test --profile localstack
$ aws configure set region ap-northeast-1 --profile localstack
```

LocalStack への接続確認：

```bash
$ aws --endpoint-url=http://localhost:4566 --profile localstack s3 ls
```

## 4. 開発用スクリプトの準備

`package.json` にスクリプトを追加：

```json
{
  "scripts": {
    "start:localstack": "docker-compose up -d",
    "stop:localstack": "docker-compose down",
    "aws:local": "aws --endpoint-url=http://localhost:4566 --profile localstack",
    "setup:resources": "node scripts/setup-resources.mjs"
  }
}
```

## 5. 初期リソースセットアップスクリプト

ArtVault で使用する基本的な AWS リソースを作成するスクリプトを準備します。

`scripts/setup-resources.mjs` を作成：

```js
import { S3Client, CreateBucketCommand } from '@aws-sdk/client-s3';
import { DynamoDBClient, CreateTableCommand } from '@aws-sdk/client-dynamodb';

// LocalStack 用の設定
const config = {
  endpoint: 'http://localhost:4566',
  region: 'ap-northeast-1',
  credentials: {
    accessKeyId: 'test',
    secretAccessKey: 'test'
  }
};

const s3Client = new S3Client({ ...config, forcePathStyle: true });
const dynamoClient = new DynamoDBClient(config);

async function setupResources() {
  try {
    console.log('🎨 ArtVault リソースをセットアップ中...');

    // S3 バケット作成（既存ならスキップ）
    try {
      await s3Client.send(new CreateBucketCommand({
        Bucket: 'artvault-gallery',
        CreateBucketConfiguration: { LocationConstraint: config.region }
      }));
      console.log('✅ S3 バケット "artvault-gallery" を作成しました');
    } catch (error) {
      if (error.name === 'BucketAlreadyOwnedByYou' || error.name === 'BucketAlreadyExists' || error.$metadata?.httpStatusCode === 409) {
        console.log('ℹ️ S3 バケット "artvault-gallery" は既に存在します。スキップします。');
      } else {
        throw error;
      }
    }

    // DynamoDB テーブル作成（既存ならスキップ）
    try {
      await dynamoClient.send(new CreateTableCommand({
        TableName: 'ArtworkTable',
        KeySchema: [
          { AttributeName: 'artworkId', KeyType: 'HASH' }
        ],
        AttributeDefinitions: [
          { AttributeName: 'artworkId', AttributeType: 'S' }
        ],
        BillingMode: 'PAY_PER_REQUEST'
      }));
      console.log('✅ DynamoDB テーブル "ArtworkTable" を作成しました');
    } catch (error) {
      if (error.name === 'ResourceInUseException' || error.$metadata?.httpStatusCode === 400) {
        console.log('ℹ️ DynamoDB テーブル "ArtworkTable" は既に存在します。スキップします。');
      } else {
        throw error;
      }
    }

    console.log('🚀 セットアップ完了！');
  } catch (error) {
    console.error('❌ セットアップエラー:', error.message);
  }
}

setupResources();
```


## 6. 動作確認

環境が正しく構築できているか確認します．もしすでに起動している場合は諸々を停止した後に実行してください：

```bash
# LocalStack 起動
$ pnpm start:localstack

# リソースセットアップ
$ pnpm setup:resources

# S3 バケット確認
$ pnpm aws:local s3 ls

# DynamoDB テーブル確認
$ pnpm aws:local dynamodb list-tables
```

## 7. README.md の作成

プロジェクトの概要を記載した README.md を作成します：

```bash
$ touch README.md
```

`README.md` の内容：

```markdown
# ArtVault Gallery

LocalStack を使った AWS サービス学習用のデジタルアートギャラリープロジェクト

## セットアップ

```bash
# LocalStack 起動
pnpm start:localstack

# AWS リソース作成
pnpm setup:resources
```

## 動作確認

```bash
# S3 バケット確認
pnpm aws:local s3 ls

# DynamoDB テーブル確認
pnpm aws:local dynamodb list-tables
```

## 8. 現在のプロジェクト構成

この章で作成されるファイル構成：

```
artvault-gallery/
├── docker-compose.yml
├── package.json
├── scripts/
│   └── setup-resources.mjs
└── README.md
```

## トラブルシューティング

### Docker が起動しない場合
- Docker Desktop が起動しているか確認
- ポート 4566 が他のプロセスで使用されていないか確認

### AWS CLI でエラーが出る場合
- プロファイル設定を再確認
- LocalStack が正常に起動しているか `docker-compose logs` で確認

## 次の章へ

環境構築が完了したら、次章では S3 を使ってアートギャラリーのフロントエンドを構築していきます。実際にブラウザで作品を表示できるギャラリーページを作成していきましょう！