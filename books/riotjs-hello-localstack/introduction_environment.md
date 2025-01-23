---
title: 'LocalStack の導入と環境構築'
---

この章では，LocalStack の概要，セットアップ方法，基本的な操作について学びます．

## LocalStack とは

[LocalStack](https://localstack.cloud/) は，AWS の主要なサービスを模擬的に提供するツールです．ローカル環境で AWS のリソースを作成，管理することができるため，学習やテストに最適です．

### 主な特徴

- ローカル環境で AWS を模擬: オフラインで動作．
- 低コスト: 実際の AWS を使わないため，コストが発生しません．
- 複数のサービスをサポート: `S3`, `DynamoDB`, `Lambda`, `API Gateway` など．

### LocalStack の制約

- 本番環境の完全な代替にはならない．
- 一部の AWS サービスや機能はサポートされていない．
- パフォーマンスや互換性に制約がある場合がある．

## LocalStack のセットアップ

概要はここまでで，実際にセットアップを始めます．

### 必要な環境

- Docker（LocalStack は Docker コンテナとして動作します）
- Python 3.8 以上
- pip

### インストール手順

1. **Docker のインストール**

LocalStack は Docker を使用して動作します．公式サイト（https://www.docker.com/ ）から Docker Desktop をインストールしてください．

2. **LocalStack のインストール**

LocalStack は Python のパッケージとして提供されています．
以下のコマンドを使用してインストールします．

```bash
> pip install localstack
```

3. **LocalStack の起動**

インストール後，以下のコマンドで LocalStack を起動します．

```bash
> localstack start
```

起動すると，サポートされている AWS サービスの一覧とエンドポイントが表示されます．

## AWS CLI との連携

LocalStack は AWS CLI と連携することで，AWS と同じように操作できます．

### AWS CLI のインストール

以下のコマンドで AWS CLI をインストールします．

```bash
> pip install awscli
```

### プロファイルの設定

LocalStack 用のプロファイルを設定します．

```bash
> aws configure --profile localstack
```

設定値は以下のように入力してください：

```
AWS Access Key ID: 任意の値（例: test）
AWS Secret Access Key: 任意の値（例: test）
Default region name: 任意のリージョン（例: us-east-1）
Default output format: 任意（例: json）
```

### エンドポイントの指定

LocalStack を利用する際は， `--endpoint-url` オプションでエンドポイントを指定します．

例: S3 バケットの作成

```bash
> aws --endpoint-url=http://localhost:4566 s3 mb s3://my-bucket --profile localstack
```

## `SDK（Boto3）` との連携

LocalStack は AWS SDK（例: Boto3）とも連携可能です．

### Boto3 のインストール

以下のコマンドで Boto3 をインストールします．

```bash
> pip install boto3
```

サンプルコード

以下は，Python で S3 バケットを作成するサンプルコードです．

```python
import boto3

# LocalStack 用のエンドポイント
endpoint_url = "http://localhost:4566"

# S3 クライアントの作成
s3 = boto3.client('s3', endpoint_url=endpoint_url)

# バケットの作成
bucket_name = "my-bucket"
s3.create_bucket(Bucket=bucket_name)
print(f"Bucket {bucket_name} created successfully!")
```

## 確認課題

1. LocalStack をインストールし，S3 バケットを CLI で作成してください．
2. Boto3 を使って S3 にファイルをアップロードしてください．
3. LocalStack がサポートするサービス一覧を確認し，興味のあるサービスをリストアップしてください．

次章では，`S3` の詳細な使い方を学び，簡単な静的ウェブホスティングを行います．
