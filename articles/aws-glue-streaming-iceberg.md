---
title: "AWS Glue StreamingでKinesisからIcebergへリアルタイム書き込み"
emoji: "🌊"
type: "tech"
topics: ["aws", "glue", "iceberg", "kinesis", "streaming"]
published: true
---

## はじめに

本記事では、AWS 環境で **Kinesis + Glue Streaming + Iceberg** を使ったストリーミングデータパイプラインを構築する方法を検証していきます。

以前、ローカル環境で Spark Structured Streaming + Kafka + Iceberg の構成を検証しました。本記事では、その構成を AWS のマネージドサービスで実装するとどうなるかを確認します。

https://zenn.dev/toshiro3/articles/spark-kafka-iceberg-docker-setup

https://zenn.dev/toshiro3/articles/spark-kafka-iceberg-streaming-basics

## ローカル環境との対応関係

| ローカル環境 | AWS 環境 |
|-------------|----------|
| Apache Kafka | Amazon Kinesis Data Streams |
| Spark Structured Streaming | AWS Glue Streaming（Spark ベース） |
| Iceberg REST Catalog | AWS Glue Data Catalog |
| MinIO（S3 互換） | Amazon S3 |

AWS Glue は内部的に Apache Spark を使用しており、Spark Structured Streaming のコードをほぼそのまま移植できます。

## アーキテクチャ

```
┌─────────────┐    ┌─────────────────┐    ┌─────────────────────────┐
│   Producer  │───▶│ Kinesis Data    │───▶│ Glue Streaming Job      │
│ (テストデータ)│    │ Streams         │    │ (Spark Structured       │
└─────────────┘    └─────────────────┘    │  Streaming)             │
                                          └───────────┬─────────────┘
                                                      │
                                                      ▼
                                          ┌─────────────────────────┐
                                          │ S3 + Glue Data Catalog  │
                                          │ (Iceberg テーブル)        │
                                          └───────────┬─────────────┘
                                                      │
                                                      ▼
                                          ┌─────────────────────────┐
                                          │ Amazon Athena           │
                                          │ (クエリ)                  │
                                          └─────────────────────────┘
```

## 前提条件

- AWS CLI がインストール・設定済み
- 適切な IAM 権限（Glue、Kinesis、S3、Athena）
- AWS リージョン: ap-northeast-1（東京）

## 環境変数の設定

```bash
export AWS_REGION="ap-northeast-1"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export PREFIX="glue-streaming-iceberg"
export KINESIS_STREAM="${PREFIX}-stream"
export S3_BUCKET_SCRIPTS="${PREFIX}-scripts-${AWS_ACCOUNT_ID}"
export S3_BUCKET_DATA="${PREFIX}-data-${AWS_ACCOUNT_ID}"
export GLUE_DATABASE="${PREFIX//-/_}_db"
export GLUE_ROLE="${PREFIX}-glue-role"
```

## Step 1: S3 バケットの作成

スクリプト用とデータ用の 2 つのバケットを作成します。

スクリプト用バケット:

```bash
aws s3 mb s3://${S3_BUCKET_SCRIPTS} --region ${AWS_REGION}
```

データ用バケット（Iceberg テーブルの保存先）:

```bash
aws s3 mb s3://${S3_BUCKET_DATA} --region ${AWS_REGION}
```

## Step 2: Kinesis Data Streams の作成

```bash
aws kinesis create-stream \
  --stream-name ${KINESIS_STREAM} \
  --stream-mode-details StreamMode=ON_DEMAND
```

ON_DEMAND モードを使用することで、シャード数を気にせずスケーリングできます。

:::message
**シャード数と Glue workers の関係**

Glue Streaming は内部的に Spark の Direct Consumer を使用しており、Kinesis のシャード数に基づいて並列度が決まります。そのため、Glue 側の `number-of-workers` を増やしても、シャード数が少ないとリソースが有効活用されません。本番環境では、想定スループットに応じてシャード数と worker 数のバランスを調整してください。
:::

## Step 3: Glue Database の作成

```bash
aws glue create-database \
  --database-input '{
    "Name": "'"${GLUE_DATABASE}"'",
    "Description": "Database for Glue Streaming Iceberg demo"
  }'
```

## Step 4: Lake Formation の設定

AWS アカウントで Lake Formation が有効になっている場合、Glue Data Catalog へのアクセスに追加の権限設定が必要です。

現在のユーザーを Lake Formation 管理者に追加:

```bash
CURRENT_USER_ARN=$(aws sts get-caller-identity --query 'Arn' --output text)
echo "Current user: ${CURRENT_USER_ARN}"

aws lakeformation put-data-lake-settings \
  --data-lake-settings '{
    "DataLakeAdmins": [
      {"DataLakePrincipalIdentifier": "'"${CURRENT_USER_ARN}"'"}
    ],
    "CreateDatabaseDefaultPermissions": [
      {
        "Principal": {"DataLakePrincipalIdentifier": "IAM_ALLOWED_PRINCIPALS"},
        "Permissions": ["ALL"]
      }
    ],
    "CreateTableDefaultPermissions": [
      {
        "Principal": {"DataLakePrincipalIdentifier": "IAM_ALLOWED_PRINCIPALS"},
        "Permissions": ["ALL"]
      }
    ]
  }'
```

Glue Database に IAM 権限を付与:

```bash
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier="IAM_ALLOWED_PRINCIPALS" \
  --resource '{"Database": {"Name": "'"${GLUE_DATABASE}"'"}}' \
  --permissions "ALL"
```

この設定により、IAM ポリシーベースのアクセス制御が有効になり、Glue Job が作成したテーブルに Athena からアクセスできるようになります。

## Step 5: IAM ロールの作成

### 信頼ポリシー

```bash
cat > glue-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "glue.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name ${GLUE_ROLE} \
  --assume-role-policy-document file://glue-trust-policy.json
```

### マネージドポリシーのアタッチ

```bash
aws iam attach-role-policy \
  --role-name ${GLUE_ROLE} \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole

aws iam attach-role-policy \
  --role-name ${GLUE_ROLE} \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

aws iam attach-role-policy \
  --role-name ${GLUE_ROLE} \
  --policy-arn arn:aws:iam::aws:policy/AmazonKinesisReadOnlyAccess
```

### カスタムポリシー

Glue Data Catalog への Iceberg メタデータ書き込み権限を追加します。

```bash
cat > glue-custom-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "GlueCatalogAccess",
      "Effect": "Allow",
      "Action": [
        "glue:GetDatabase",
        "glue:GetDatabases",
        "glue:CreateTable",
        "glue:GetTable",
        "glue:GetTables",
        "glue:UpdateTable",
        "glue:DeleteTable",
        "glue:GetPartition",
        "glue:GetPartitions",
        "glue:CreatePartition",
        "glue:BatchCreatePartition",
        "glue:DeletePartition"
      ],
      "Resource": [
        "arn:aws:glue:${AWS_REGION}:${AWS_ACCOUNT_ID}:catalog",
        "arn:aws:glue:${AWS_REGION}:${AWS_ACCOUNT_ID}:database/${GLUE_DATABASE}",
        "arn:aws:glue:${AWS_REGION}:${AWS_ACCOUNT_ID}:table/${GLUE_DATABASE}/*"
      ]
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name ${GLUE_ROLE} \
  --policy-name "${GLUE_ROLE}-custom-policy" \
  --policy-document file://glue-custom-policy.json
```

## Step 6: Glue Streaming スクリプトの作成

ローカル環境の Spark Structured Streaming コードをベースに、AWS 環境用に修正します。

```bash
cat > glue_streaming_iceberg.py << 'EOF'
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.sql.functions import from_json, col, current_timestamp
from pyspark.sql.types import StructType, StructField, StringType, TimestampType

# ジョブパラメータ取得
args = getResolvedOptions(sys.argv, [
    'JOB_NAME',
    'kinesis_stream_name',
    'database_name',
    'table_name',
    's3_warehouse_path'
])

# SparkContext / GlueContext 初期化
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# パラメータ
kinesis_stream_name = args['kinesis_stream_name']
database_name = args['database_name']
table_name = args['table_name']
s3_warehouse_path = args['s3_warehouse_path']
region = "ap-northeast-1"

# Iceberg カタログ設定（Glue Data Catalog 使用）
# Athena のデフォルトカタログ名と一致させることで、クエリ時の指定が簡潔になる
catalog_name = "awsdatacatalog"
spark.conf.set(f"spark.sql.catalog.{catalog_name}", "org.apache.iceberg.spark.SparkCatalog")
spark.conf.set(f"spark.sql.catalog.{catalog_name}.warehouse", s3_warehouse_path)
spark.conf.set(f"spark.sql.catalog.{catalog_name}.catalog-impl", "org.apache.iceberg.aws.glue.GlueCatalog")
spark.conf.set(f"spark.sql.catalog.{catalog_name}.io-impl", "org.apache.iceberg.aws.s3.S3FileIO")

# テーブル作成（存在しない場合）
full_table_name = f"{catalog_name}.{database_name}.{table_name}"
spark.sql(f"""
    CREATE TABLE IF NOT EXISTS {full_table_name} (
        event_id STRING,
        user_id STRING,
        event_type STRING,
        page STRING,
        event_time TIMESTAMP,
        processed_time TIMESTAMP
    )
    USING iceberg
""")

print(f"Table {full_table_name} is ready")

# JSON スキーマ定義（入力データの形式）
schema = StructType([
    StructField("event_id", StringType()),
    StructField("user_id", StringType()),
    StructField("event_type", StringType()),
    StructField("page", StringType()),
    StructField("timestamp", StringType())
])

# Kinesis からストリーミング読み込み
kinesis_df = spark.readStream \
    .format("kinesis") \
    .option("streamName", kinesis_stream_name) \
    .option("endpointUrl", f"https://kinesis.{region}.amazonaws.com") \
    .option("startingPosition", "TRIM_HORIZON") \
    .load()

# データ変換
# - Kinesis の data カラムは Base64 エンコードされているため文字列にキャスト
# - JSON をパースして構造化
# - タイムスタンプ型に変換
parsed_df = kinesis_df \
    .selectExpr("CAST(data AS STRING) as json_data") \
    .select(from_json(col("json_data"), schema).alias("data")) \
    .select(
        col("data.event_id"),
        col("data.user_id"),
        col("data.event_type"),
        col("data.page"),
        col("data.timestamp").cast(TimestampType()).alias("event_time"),
        current_timestamp().alias("processed_time")
    )

# チェックポイント用パス
checkpoint_path = f"{s3_warehouse_path}/checkpoints/{table_name}"

# Iceberg テーブルへストリーミング書き込み
query = parsed_df.writeStream \
    .format("iceberg") \
    .outputMode("append") \
    .option("checkpointLocation", checkpoint_path) \
    .toTable(full_table_name)

query.awaitTermination()

job.commit()
EOF
```

### ローカル環境との違い

| 項目 | ローカル環境 | AWS 環境 |
|------|-------------|----------|
| ストリームソース | `.format("kafka")` | `.format("kinesis")` |
| カタログ実装 | `RESTCatalog` | `GlueCatalog` |
| ストレージ | MinIO（S3 互換） | Amazon S3 |
| チェックポイント | ローカルファイルシステム | S3 |

基本的なデータ変換ロジック（JSON パース、型変換）はほぼ同じです。

## Step 7: スクリプトのアップロードと Job 作成

スクリプトを S3 にアップロード:

```bash
aws s3 cp glue_streaming_iceberg.py s3://${S3_BUCKET_SCRIPTS}/scripts/
```

Glue Streaming Job を作成:

```bash
aws glue create-job \
  --name "${PREFIX}-job" \
  --role "arn:aws:iam::${AWS_ACCOUNT_ID}:role/${GLUE_ROLE}" \
  --command '{
    "Name": "gluestreaming",
    "ScriptLocation": "s3://'"${S3_BUCKET_SCRIPTS}"'/scripts/glue_streaming_iceberg.py",
    "PythonVersion": "3"
  }' \
  --default-arguments '{
    "--job-language": "python",
    "--datalake-formats": "iceberg",
    "--kinesis_stream_name": "'"${KINESIS_STREAM}"'",
    "--database_name": "'"${GLUE_DATABASE}"'",
    "--table_name": "streaming_events",
    "--s3_warehouse_path": "s3://'"${S3_BUCKET_DATA}"'/warehouse"
  }' \
  --glue-version "5.0" \
  --number-of-workers 2 \
  --worker-type "G.1X"
```

**ポイント**:
- `--glue-version "5.0"`: 2024年12月に一般提供開始された最新バージョン
- `--datalake-formats iceberg`: Iceberg サポートを有効化
- `gluestreaming`: Streaming Job タイプを指定
- `G.1X x 2 workers`: 最小構成（検証用）

## Step 8: Job の実行

```bash
aws glue start-job-run --job-name "${PREFIX}-job"
```

ジョブの状態を確認（RUNNING になるまで 2〜3 分かかります）:

```bash
while true; do
  STATE=$(aws glue get-job-runs --job-name "${PREFIX}-job" \
    --query 'JobRuns[0].JobRunState' --output text)
  echo "Job state: ${STATE}"
  if [ "$STATE" = "RUNNING" ]; then
    break
  fi
  sleep 10
done
```

## Step 9: テストデータの送信

テストデータを Kinesis に送信するスクリプトを作成します。

```bash
cat > send_test_events.sh << 'SCRIPT'
#!/bin/bash
EVENT_TYPES=("click" "page_view" "purchase")

for i in {1..10}; do
  EVENT_TYPE=${EVENT_TYPES[$((RANDOM % 3))]}
  DATA=$(cat << EOF
{
  "event_id": "evt_$(date +%s)_${i}",
  "user_id": "user_00$((RANDOM % 5))",
  "event_type": "${EVENT_TYPE}",
  "page": "/products/$((100 + i))",
  "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
}
EOF
)
  
  aws kinesis put-record \
    --stream-name ${KINESIS_STREAM} \
    --partition-key "pk_${i}" \
    --data "$(echo -n ${DATA} | base64)"
  
  echo "Sent event ${i}: ${EVENT_TYPE}"
  sleep 2
done
SCRIPT
```

実行:

```bash
chmod +x send_test_events.sh
./send_test_events.sh
```

## Step 10: Athena でデータを確認

2〜3 分待ってから、Athena でデータを確認します。

```bash
QUERY_ID=$(aws athena start-query-execution \
  --query-string "SELECT * FROM ${GLUE_DATABASE}.streaming_events ORDER BY event_time DESC LIMIT 10" \
  --result-configuration "OutputLocation=s3://${S3_BUCKET_DATA}/athena-results/" \
  --work-group "primary" \
  --query 'QueryExecutionId' --output text)

echo "Query ID: ${QUERY_ID}"
echo "Waiting for query to complete..."
sleep 5

aws athena get-query-results --query-execution-id ${QUERY_ID} \
  --query 'ResultSet.Rows[*].Data[*].VarCharValue'
```

出力例:

```json
[
    ["event_id", "user_id", "event_type", "page", "event_time", "processed_time"],
    ["evt_1767884412_10", "user_000", "click", "/products/110", "2026-01-08T15:16:52Z", "2026-01-08T15:17:01Z"],
    ["evt_1767884410_9", "user_004", "page_view", "/products/109", "2026-01-08T15:16:50Z", "2026-01-08T15:17:01Z"],
    ...
]
```

## Step 11: Job の停止とクリーンアップ

### Streaming Job の停止

```bash
JOB_RUN_ID=$(aws glue get-job-runs --job-name "${PREFIX}-job" \
  --query 'JobRuns[?JobRunState==`RUNNING`].Id' --output text)

if [ -n "$JOB_RUN_ID" ]; then
  aws glue batch-stop-job-run \
    --job-name "${PREFIX}-job" \
    --job-run-ids ${JOB_RUN_ID}
fi
```

### リソースの削除

Glue Job:

```bash
aws glue delete-job --job-name "${PREFIX}-job"
```

Kinesis Stream:

```bash
aws kinesis delete-stream --stream-name ${KINESIS_STREAM}
```

Glue Table と Database:

```bash
aws glue delete-table --database-name ${GLUE_DATABASE} --name "streaming_events"
aws glue delete-database --name ${GLUE_DATABASE}
```

S3 バケット:

```bash
aws s3 rm s3://${S3_BUCKET_SCRIPTS}/ --recursive
aws s3 rb s3://${S3_BUCKET_SCRIPTS}
aws s3 rm s3://${S3_BUCKET_DATA}/ --recursive
aws s3 rb s3://${S3_BUCKET_DATA}
```

IAM ロール:

```bash
aws iam detach-role-policy --role-name ${GLUE_ROLE} \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole
aws iam detach-role-policy --role-name ${GLUE_ROLE} \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
aws iam detach-role-policy --role-name ${GLUE_ROLE} \
  --policy-arn arn:aws:iam::aws:policy/AmazonKinesisReadOnlyAccess
aws iam delete-role-policy --role-name ${GLUE_ROLE} \
  --policy-name "${GLUE_ROLE}-custom-policy"
aws iam delete-role --role-name ${GLUE_ROLE}
```

## ローカル環境との比較

### 共通点

| 項目 | 説明 |
|------|------|
| 処理エンジン | Apache Spark（Structured Streaming） |
| テーブルフォーマット | Apache Iceberg |
| 書き込み方式 | `.writeStream.format("iceberg").toTable()` |
| チェックポイント | 障害復旧のためのオフセット管理 |

### 相違点

| 項目 | ローカル環境 | AWS 環境 |
|------|-------------|----------|
| 起動時間 | 数秒 | 2〜3 分（クラスタ起動） |
| スケーリング | 手動 | 自動（DPU 数で調整） |
| メタデータ管理 | REST Catalog（手動管理） | Glue Data Catalog（マネージド） |
| 運用コスト | 無料 | DPU 時間課金 |
| クエリエンジン | Spark SQL / Trino | Athena / Redshift Spectrum |

### コストに関する注意

Glue Streaming Job は **実行時間に応じて課金** されます。検証後は必ず Job を停止してください。

- G.1X ワーカー: 約 $0.44/DPU-hour
- 2 ワーカー × 1 時間 = 約 $0.88

## トラブルシューティング

### Job が FAILED になる

1. CloudWatch Logs でエラーを確認:

```bash
aws logs describe-log-streams \
  --log-group-name "/aws-glue/jobs/output" \
  --order-by LastEventTime --descending --limit 1
```

2. よくある原因:
   - IAM 権限不足（S3、Kinesis、Glue Catalog）
   - Kinesis Stream が存在しない
   - S3 パスの指定ミス

### データが書き込まれない

1. Kinesis にデータが到達しているか確認:

```bash
aws kinesis get-shard-iterator \
  --stream-name ${KINESIS_STREAM} \
  --shard-id shardId-000000000000 \
  --shard-iterator-type TRIM_HORIZON

aws kinesis get-records --shard-iterator <iterator>
```

2. チェックポイントをクリアして再実行:

```bash
aws s3 rm s3://${S3_BUCKET_DATA}/warehouse/checkpoints/ --recursive
```

:::message alert
**注意**: チェックポイントを削除すると、次回 Job 実行時に Kinesis の最初（TRIM_HORIZON）から再読み込みされます。これにより、既に処理済みのデータが再度書き込まれ、**データの重複が発生**する可能性があります。本番環境では慎重に判断してください。
:::

### Lake Formation 権限エラー

Athena クエリで `Insufficient Lake Formation permission(s)` エラーが発生する場合:

現在のユーザーを Lake Formation 管理者に追加:

```bash
CURRENT_USER_ARN=$(aws sts get-caller-identity --query 'Arn' --output text)

aws lakeformation put-data-lake-settings \
  --data-lake-settings '{
    "DataLakeAdmins": [
      {"DataLakePrincipalIdentifier": "'"${CURRENT_USER_ARN}"'"}
    ],
    "CreateDatabaseDefaultPermissions": [
      {
        "Principal": {"DataLakePrincipalIdentifier": "IAM_ALLOWED_PRINCIPALS"},
        "Permissions": ["ALL"]
      }
    ],
    "CreateTableDefaultPermissions": [
      {
        "Principal": {"DataLakePrincipalIdentifier": "IAM_ALLOWED_PRINCIPALS"},
        "Permissions": ["ALL"]
      }
    ]
  }'
```

テーブルへの権限を付与:

```bash
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier="IAM_ALLOWED_PRINCIPALS" \
  --resource '{"Table": {"DatabaseName": "'"${GLUE_DATABASE}"'", "Name": "streaming_events"}}' \
  --permissions "ALL"
```

## まとめ

本記事では、ローカル環境で検証した Spark Structured Streaming + Kafka + Iceberg の構成を、AWS 環境（Kinesis + Glue Streaming + Iceberg）で実装しました。

**検証できたこと**:
- Kinesis Data Streams からのストリーミング読み込み
- Glue Streaming Job での Spark Structured Streaming 実行
- Iceberg テーブルへのリアルタイム書き込み
- Glue Data Catalog によるメタデータ管理
- Athena でのクエリ実行

**ローカル環境との違い**:
- ストリームソースが Kafka から Kinesis に変更
- カタログが REST Catalog から Glue Data Catalog に変更
- マネージドサービスによる運用負荷の軽減

ローカル環境で培った Spark Structured Streaming の知識は、そのまま AWS 環境でも活用できます。

## 補足: Iceberg テーブルのメンテナンス

ストリーミング書き込みを続けると、小さなデータファイルやメタデータファイルが大量に作成されます。これにより Athena のクエリパフォーマンスが低下する可能性があります。

本番環境では、以下のメンテナンス処理を定期的に実行することを推奨します:

- **OPTIMIZE（コンパクション）**: 小さなファイルを統合して大きなファイルにまとめる
- **VACUUM**: 不要になった古いスナップショットやデータファイルを削除する

```sql
-- コンパクション（Athena で実行）
OPTIMIZE awsdatacatalog.glue_streaming_iceberg_db.streaming_events REWRITE DATA USING BIN_PACK;

-- 古いスナップショットの削除（7日以上前のものを削除）
VACUUM awsdatacatalog.glue_streaming_iceberg_db.streaming_events;
```

これらの処理は、EventBridge + Lambda や Step Functions で定期実行するのが一般的です。

## 補足: S3 Tables について

AWS では 2024 年 12 月に **Amazon S3 Tables** という新しい機能がリリースされました。S3 Tables は Iceberg テーブルのメタデータを自動管理し、コンパクションも自動化されます。

今度検証してみようと思います。

## 参考リンク

- [AWS Glue Streaming ETL](https://docs.aws.amazon.com/glue/latest/dg/add-job-streaming.html)
- [Using the Iceberg framework in AWS Glue](https://docs.aws.amazon.com/glue/latest/dg/aws-glue-programming-etl-format-iceberg.html)
- [GitHub - aws-samples/aws-glue-streaming-etl-with-apache-iceberg](https://github.com/aws-samples/aws-glue-streaming-etl-with-apache-iceberg)
