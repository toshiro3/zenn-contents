---
title: "DynamoDB Streams → Firehose → S3 → Athena のニアリアルタイムデータパイプライン構築ガイド"
emoji: "🔥"
type: "tech"
topics: ["aws", "dynamodb", "firehose", "athena", "eventbridge"]
published: false
---

## はじめに

DynamoDB のデータをニアリアルタイムで分析基盤に連携したいケースは多いと思います。本記事では、DynamoDB Streams から Firehose 経由で S3 に出力し、Athena でクエリできる構成を実際に構築・検証しました。

2つの実装パターンを比較しながら、それぞれのメリット・デメリットとコスト特性を解説します。

## 対象読者

- DynamoDB のデータを分析用途で活用したい方
- ニアリアルタイムのデータパイプラインを構築したい方
- EventBridge Pipes と Lambda の使い分けを知りたい方

## アーキテクチャ

### 全体構成

```
DynamoDB (Streams有効)
    ↓
┌─────────────────────────────────────┐
│ パターン1: EventBridge Pipes       │
│ パターン2: Lambda + Event Source   │
│            Mapping                 │
└─────────────────────────────────────┘
    ↓
Firehose (バッファリング + 圧縮)
    ↓
S3 (JSON Lines, Hiveパーティション)
    ↓
Athena (Partition Projection)
```

### 重要なポイント

**DynamoDB Streams → Firehose は直接接続できません。** 間に EventBridge Pipes または Lambda を挟む必要があります。

## 前提条件

- AWS CLI がインストール・設定済み
- 適切な IAM 権限があること

---

## パターン1: EventBridge Pipes 経由

コードを書かずにシンプルに構築できる方法です。

### Step 1: DynamoDB テーブル作成（Streams有効）

```bash
aws dynamodb create-table \
  --table-name ViewerLogs \
  --attribute-definitions \
    AttributeName=stream_id,AttributeType=S \
    AttributeName=timestamp,AttributeType=S \
  --key-schema \
    AttributeName=stream_id,KeyType=HASH \
    AttributeName=timestamp,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_IMAGE \
  --region ap-northeast-1
```

StreamViewType の選択肢：

| 値 | 内容 | ユースケース |
|---|---|---|
| `NEW_IMAGE` | 変更後のアイテム全体 | 分析向け |
| `OLD_IMAGE` | 変更前のアイテム全体 | 監査ログ |
| `NEW_AND_OLD_IMAGES` | 両方 | 差分検知 |
| `KEYS_ONLY` | キーのみ | 軽量通知 |

:::message alert
**同一アイテムへの複数操作について**

DynamoDB Streams は INSERT / MODIFY / REMOVE の各操作ごとにイベントを発行します。同じアイテムに対して「追加 → 更新 → 更新 → 削除」のような操作があった場合、S3 には複数のレコードが保存されます。

本記事の構成は raw データ（変更履歴）の保存に適しています。最新状態のみが必要な場合は、dbt などを使って staging / mart レイヤーで重複排除や集計を行う構成を検討してください。
:::

Stream ARN を取得します：

```bash
aws dynamodb describe-table --table-name ViewerLogs \
  --query 'Table.LatestStreamArn' --output text \
  --region ap-northeast-1
```

### Step 2: S3 バケット作成

```bash
# バケット名はアカウントIDなどを含めて一意にする
aws s3 mb s3://viewer-logs-firehose-<YOUR_ACCOUNT_ID> \
  --region ap-northeast-1
```

### Step 3: Firehose 用 IAM ロール作成

```bash
# 信頼ポリシー
cat << 'EOF' > firehose-trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "firehose.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name FirehoseToS3Role \
  --assume-role-policy-document file://firehose-trust-policy.json
```

```bash
# S3書き込み権限（バケット名は適宜変更）
cat << 'EOF' > firehose-s3-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::viewer-logs-firehose-<YOUR_ACCOUNT_ID>",
        "arn:aws:s3:::viewer-logs-firehose-<YOUR_ACCOUNT_ID>/*"
      ]
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name FirehoseToS3Role \
  --policy-name S3WriteAccess \
  --policy-document file://firehose-s3-policy.json
```

### Step 4: Firehose Delivery Stream 作成

```bash
aws firehose create-delivery-stream \
  --delivery-stream-name viewer-logs-stream \
  --delivery-stream-type DirectPut \
  --s3-destination-configuration '{
    "RoleARN": "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/FirehoseToS3Role",
    "BucketARN": "arn:aws:s3:::viewer-logs-firehose-<YOUR_ACCOUNT_ID>",
    "Prefix": "raw/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/",
    "ErrorOutputPrefix": "errors/",
    "BufferingHints": {
      "SizeInMBs": 1,
      "IntervalInSeconds": 60
    },
    "CompressionFormat": "GZIP"
  }' \
  --region ap-northeast-1
```

**バッファ設定の目安：**

| ユースケース | IntervalInSeconds | SizeInMBs |
|---|---|---|
| リアルタイムダッシュボード | 60（最小） | 1 |
| 準リアルタイム分析 | 300 | 5 |
| バッチ分析 | 900（最大） | 128 |

### Step 5: EventBridge Pipes 用 IAM ロール作成

```bash
cat << 'EOF' > pipes-trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "pipes.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name EventBridgePipesRole \
  --assume-role-policy-document file://pipes-trust-policy.json
```

```bash
cat << 'EOF' > pipes-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:DescribeStream",
        "dynamodb:GetRecords",
        "dynamodb:GetShardIterator",
        "dynamodb:ListStreams"
      ],
      "Resource": "arn:aws:dynamodb:ap-northeast-1:<YOUR_ACCOUNT_ID>:table/ViewerLogs/stream/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "firehose:PutRecord",
        "firehose:PutRecordBatch"
      ],
      "Resource": "arn:aws:firehose:ap-northeast-1:<YOUR_ACCOUNT_ID>:deliverystream/viewer-logs-stream"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name EventBridgePipesRole \
  --policy-name PipesAccess \
  --policy-document file://pipes-policy.json
```

### Step 6: EventBridge Pipes 作成

DynamoDB Streams のデータはそのままだと分析に不向きなため、`InputTemplate` で必要なフィールドだけ抽出し、JSON Lines 形式に変換します。

```bash
aws pipes create-pipe \
  --name dynamodb-to-firehose \
  --source "arn:aws:dynamodb:ap-northeast-1:<YOUR_ACCOUNT_ID>:table/ViewerLogs/stream/<YOUR_STREAM_TIMESTAMP>" \
  --target "arn:aws:firehose:ap-northeast-1:<YOUR_ACCOUNT_ID>:deliverystream/viewer-logs-stream" \
  --role-arn "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/EventBridgePipesRole" \
  --source-parameters '{
    "DynamoDBStreamParameters": {
      "StartingPosition": "LATEST",
      "BatchSize": 10
    }
  }' \
  --target-parameters '{
    "InputTemplate": "{\"event_id\": \"<$.eventID>\", \"event_name\": \"<$.eventName>\", \"stream_id\": \"<$.dynamodb.NewImage.stream_id.S>\", \"timestamp\": \"<$.dynamodb.NewImage.timestamp.S>\", \"user_id\": \"<$.dynamodb.NewImage.user_id.S>\", \"action\": \"<$.dynamodb.NewImage.action.S>\", \"device\": \"<$.dynamodb.NewImage.device.S>\"}\n"
  }' \
  --region ap-northeast-1
```

:::message
**InputTemplate のポイント**
- `<$.dynamodb.NewImage.xxx.S>` で DynamoDB の型情報を除去しながら必要なフィールドを抽出
- 末尾に `\n` を追加して JSON Lines 形式に（Athena でクエリ可能にするため）
:::

### Step 7: Athena 用の設定

```bash
# クエリ結果用バケット
aws s3 mb s3://athena-results-<YOUR_ACCOUNT_ID> \
  --region ap-northeast-1

# Glue データベース
aws glue create-database \
  --database-input '{"Name": "viewer_logs_db"}' \
  --region ap-northeast-1
```

### Step 8: Glue テーブル作成（Partition Projection）

Partition Projection を使うことで、`MSCK REPAIR TABLE` を実行しなくても新しいパーティションが自動認識されます。

```bash
aws glue create-table \
  --database-name viewer_logs_db \
  --table-input '{
    "Name": "viewer_logs",
    "StorageDescriptor": {
      "Columns": [
        {"Name": "event_id", "Type": "string"},
        {"Name": "event_name", "Type": "string"},
        {"Name": "stream_id", "Type": "string"},
        {"Name": "timestamp", "Type": "string"},
        {"Name": "user_id", "Type": "string"},
        {"Name": "action", "Type": "string"},
        {"Name": "device", "Type": "string"}
      ],
      "Location": "s3://viewer-logs-firehose-<YOUR_ACCOUNT_ID>/raw/",
      "InputFormat": "org.apache.hadoop.mapred.TextInputFormat",
      "OutputFormat": "org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat",
      "SerdeInfo": {
        "SerializationLibrary": "org.openx.data.jsonserde.JsonSerDe"
      },
      "Compressed": true
    },
    "PartitionKeys": [
      {"Name": "year", "Type": "string"},
      {"Name": "month", "Type": "string"},
      {"Name": "day", "Type": "string"}
    ],
    "TableType": "EXTERNAL_TABLE",
    "Parameters": {
      "projection.enabled": "true",
      "projection.year.type": "integer",
      "projection.year.range": "2025,2030",
      "projection.month.type": "integer",
      "projection.month.range": "1,12",
      "projection.month.digits": "2",
      "projection.day.type": "integer",
      "projection.day.range": "1,31",
      "projection.day.digits": "2",
      "storage.location.template": "s3://viewer-logs-firehose-<YOUR_ACCOUNT_ID>/raw/year=${year}/month=${month}/day=${day}/"
    }
  }' \
  --region ap-northeast-1
```

### 動作確認

テストデータを投入：

```bash
aws dynamodb put-item \
  --table-name ViewerLogs \
  --item '{
    "stream_id": {"S": "stream-001"},
    "timestamp": {"S": "2025-12-25T15:30:00Z"},
    "user_id": {"S": "user-123"},
    "action": {"S": "join"},
    "device": {"S": "iOS"}
  }' \
  --region ap-northeast-1
```

約1〜2分後に S3 を確認し、Athena でクエリを実行：

```sql
SELECT * FROM viewer_logs_db.viewer_logs LIMIT 10;
```

---

## パターン2: Lambda + Event Source Mapping 経由

より柔軟なデータ変換や、コスト調整が必要になった場合の選択肢です。

### EventBridge Pipes との違い

| 項目 | EventBridge Pipes | Lambda |
|---|---|---|
| 設定の簡単さ | ◎ コード不要 | △ コード必要 |
| データ変換 | InputTemplate のみ | 自由に実装可能 |
| レコード結合 | できない | できる |
| batching-window | なし | あり（最大300秒） |

### Firehose の5KB切り上げルールについて

Firehose は **5KB 単位で切り上げて課金** されます。この切り上げは **API オペレーション単位ではなくレコード単位** で適用されます。例えば、PutRecordBatch で 1KB のレコードを2件送信した場合、課金対象は 10KB（5KB × 2レコード）となります。

EventBridge Pipes はバッチ送信（PutRecordBatch）をサポートしていますが、InputTemplate は各レコードに個別適用されるため、複数イベントを1つのレコードに結合することはできません。そのため、小さいレコードが大量にある場合はコストが膨らむ可能性があります。

Lambda を使うと、複数のイベントを1つのレコード（文字列）にまとめてから送信できるため、Firehose から見たレコード数を減らしてコストを調整できる可能性があります。ただし、Lambda 自体の実行コストもかかるため、実際の効果は運用しながら確認する必要があります。

**参考:**
- [Amazon Data Firehose FAQs - Pricing and billing](https://aws.amazon.com/firehose/faqs/) - PutRecordBatch での5KB切り上げの説明
- [Amazon EventBridge Pipes batching and concurrency](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-pipes-batching-concurrency.html) - InputTemplate が各レコードに個別適用される説明

### Step 1: EventBridge Pipes を削除（パターン1から移行する場合）

```bash
aws pipes delete-pipe \
  --name dynamodb-to-firehose \
  --region ap-northeast-1
```

### Step 2: Lambda 用 IAM ロール作成

```bash
cat << 'EOF' > lambda-trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name LambdaDynamoDBStreamRole \
  --assume-role-policy-document file://lambda-trust-policy.json
```

```bash
cat << 'EOF' > lambda-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:DescribeStream",
        "dynamodb:GetRecords",
        "dynamodb:GetShardIterator",
        "dynamodb:ListStreams"
      ],
      "Resource": "arn:aws:dynamodb:ap-northeast-1:<YOUR_ACCOUNT_ID>:table/ViewerLogs/stream/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "firehose:PutRecord",
        "firehose:PutRecordBatch"
      ],
      "Resource": "arn:aws:firehose:ap-northeast-1:<YOUR_ACCOUNT_ID>:deliverystream/viewer-logs-stream"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name LambdaDynamoDBStreamRole \
  --policy-name DynamoDBStreamAccess \
  --policy-document file://lambda-policy.json
```

### Step 3: Lambda 関数コード

```python:index.py
import json
import boto3

firehose = boto3.client('firehose')
STREAM_NAME = 'viewer-logs-stream'

def handler(event, context):
    """
    DynamoDB Streams から複数レコードを受け取り、
    結合して1つのレコードとして Firehose に送信
    """
    
    print(f"Received {len(event['Records'])} records")
    
    lines = []
    for record in event['Records']:
        event_name = record['eventName']
        
        # DELETE は NewImage がないのでスキップ
        if 'NewImage' not in record['dynamodb']:
            continue
        
        new_image = record['dynamodb']['NewImage']
        
        # DynamoDB の型情報を除去してシンプルな dict に変換
        item = {
            'event_id': record['eventID'],
            'event_name': event_name,
            'stream_id': new_image.get('stream_id', {}).get('S'),
            'timestamp': new_image.get('timestamp', {}).get('S'),
            'user_id': new_image.get('user_id', {}).get('S'),
            'action': new_image.get('action', {}).get('S'),
            'device': new_image.get('device', {}).get('S'),
        }
        
        lines.append(json.dumps(item))
    
    if not lines:
        print("No records to process")
        return {'statusCode': 200, 'body': 'No records to process'}
    
    # 複数行を改行で結合して1レコードとして送信
    combined_record = '\n'.join(lines) + '\n'
    
    response = firehose.put_record(
        DeliveryStreamName=STREAM_NAME,
        Record={'Data': combined_record.encode('utf-8')}
    )
    
    print(f"Sent {len(lines)} events as 1 Firehose record")
    
    return {'statusCode': 200, 'body': f'Processed {len(lines)} records'}
```

### Step 4: Lambda デプロイ

```bash
zip function.zip index.py

# IAMロール反映待ち
sleep 10

aws lambda create-function \
  --function-name DynamoDBStreamProcessor \
  --runtime python3.12 \
  --handler index.handler \
  --role arn:aws:iam::<YOUR_ACCOUNT_ID>:role/LambdaDynamoDBStreamRole \
  --zip-file fileb://function.zip \
  --timeout 60 \
  --region ap-northeast-1
```

### Step 5: Event Source Mapping 作成

```bash
aws lambda create-event-source-mapping \
  --function-name DynamoDBStreamProcessor \
  --event-source-arn "arn:aws:dynamodb:ap-northeast-1:<YOUR_ACCOUNT_ID>:table/ViewerLogs/stream/<YOUR_STREAM_TIMESTAMP>" \
  --starting-position LATEST \
  --batch-size 100 \
  --maximum-batching-window-in-seconds 30 \
  --region ap-northeast-1
```

**パラメータ解説：**

| パラメータ | 説明 |
|---|---|
| `batch-size` | Lambda 1回で受け取る最大レコード数（最大1000） |
| `maximum-batching-window-in-seconds` | 最大待機時間（0〜300秒） |

:::message
**batching-window の効果**

イベントがまばらに発生する場合でも、指定した秒数までバッファリングしてから Lambda を起動します。これにより、複数イベントをまとめて処理できます。
:::

### 動作確認

```bash
# テストデータを連続投入
for i in {1..5}; do
  aws dynamodb put-item \
    --table-name ViewerLogs \
    --item "{
      \"stream_id\": {\"S\": \"stream-lambda-test\"},
      \"timestamp\": {\"S\": \"2025-12-25T17:00:0${i}Z\"},
      \"user_id\": {\"S\": \"user-${i}00\"},
      \"action\": {\"S\": \"join\"},
      \"device\": {\"S\": \"iOS\"}
    }" \
    --region ap-northeast-1
  echo "Inserted record $i"
done
```

Lambda のログを確認：

```bash
aws logs tail /aws/lambda/DynamoDBStreamProcessor \
  --since 5m \
  --region ap-northeast-1
```

以下のようなログが出れば成功です：

```
Received 5 records
Sent 5 events as 1 Firehose record
```

---

## コスト特性

### 各サービスの課金（東京リージョン）

| サービス | 課金単位 | 備考 |
|---|---|---|
| DynamoDB Streams | 毎月最初の250万リクエストは無料、以降10万件あたり $0.0228 | |
| EventBridge Pipes | 100万リクエストあたり $0.50 | 64KBチャンクごとに1リクエスト |
| Lambda | リクエスト + 実行時間 | |
| Firehose | 取り込み 1GB あたり $0.036（最初の500TB/月） | **5KB単位で切り上げ** |
| S3 | ストレージ + リクエスト | |
| Athena | スキャン 1TB あたり $5 | |

※ 料金は変更される可能性があるため、最新情報は各サービスの料金ページをご確認ください。

### コスト調整の可能性

Lambda + Event Source Mapping を使うと、`batching-window` で複数イベントをまとめて処理し、Firehose へのレコード数を減らせる可能性があります。

ただし、以下の点を考慮する必要があります：

- Lambda 自体の実行コストがかかる
- batching-window を長くするとレイテンシが増加する
- 実際の効果はイベントの発生頻度やサイズに依存する

コスト調整が必要になった場合の選択肢として、Lambda への移行を検討するのが良さそうです。

---

## パーティション管理

### Partition Projection vs Glue Crawler

| 方法 | リアルタイム性 | 追加コスト | 複雑さ |
|---|---|---|---|
| **Partition Projection** | ◎ 即時 | なし | シンプル |
| Glue Crawler | △ スケジュール依存 | あり | 中程度 |
| MSCK REPAIR TABLE | × 手動 | なし | 運用負荷大 |

特別な理由がなければ **Partition Projection** がおすすめです。

---

## まとめ

| 観点 | EventBridge Pipes | Lambda |
|---|---|---|
| 構築の容易さ | ◎ | △ |
| 柔軟性 | △ | ◎ |
| コスト調整 | △ 調整しにくい | ◎ batching-window で調整可能 |

### 推奨アプローチ

1. **まずは EventBridge Pipes で構築** → シンプルに素早くリリース
2. **運用しながら様子を見る** → CloudWatch で Firehose の IncomingRecords などを確認
3. **必要に応じて Lambda への移行を検討** → コスト調整や複雑な変換が必要になった場合

---

## クリーンアップ

検証が終わったらリソースを削除します：

```bash
# EventBridge Pipes
aws pipes delete-pipe --name dynamodb-to-firehose --region ap-northeast-1

# Lambda（パターン2を使った場合）
aws lambda delete-event-source-mapping --uuid <UUID> --region ap-northeast-1
aws lambda delete-function --function-name DynamoDBStreamProcessor --region ap-northeast-1

# Firehose
aws firehose delete-delivery-stream --delivery-stream-name viewer-logs-stream --region ap-northeast-1

# DynamoDB
aws dynamodb delete-table --table-name ViewerLogs --region ap-northeast-1

# S3
aws s3 rb s3://viewer-logs-firehose-<YOUR_ACCOUNT_ID> --force
aws s3 rb s3://athena-results-<YOUR_ACCOUNT_ID> --force

# Glue
aws glue delete-table --database-name viewer_logs_db --name viewer_logs --region ap-northeast-1
aws glue delete-database --name viewer_logs_db --region ap-northeast-1

# IAM ロール
aws iam delete-role-policy --role-name FirehoseToS3Role --policy-name S3WriteAccess
aws iam delete-role --role-name FirehoseToS3Role
aws iam delete-role-policy --role-name EventBridgePipesRole --policy-name PipesAccess
aws iam delete-role --role-name EventBridgePipesRole
aws iam delete-role-policy --role-name LambdaDynamoDBStreamRole --policy-name DynamoDBStreamAccess
aws iam delete-role --role-name LambdaDynamoDBStreamRole
```

---

## 参考リンク

- [Amazon DynamoDB Streams](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html)
- [Amazon EventBridge Pipes](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-pipes.html)
- [Amazon Data Firehose](https://docs.aws.amazon.com/firehose/latest/dev/what-is-this-service.html)
- [Partition Projection with Amazon Athena](https://docs.aws.amazon.com/athena/latest/ug/partition-projection.html)
