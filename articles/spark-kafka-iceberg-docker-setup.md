---
title: "Spark Structured Streaming + Kafka + Iceberg 環境構築（Docker Compose）"
emoji: "🐳"
type: "tech"
topics: ["spark", "kafka", "iceberg", "docker", "streaming"]
published: true
---

## はじめに

この記事では、**Spark Structured Streaming + Kafka + Apache Iceberg** を使ったストリーミングデータパイプラインのローカル検証環境を Docker Compose で構築します。

Kafka からリアルタイムでデータを読み込み、Iceberg テーブルに書き込むパイプラインを構築するための第一歩として、まずは環境構築と動作確認を行います。

## なぜ Spark Structured Streaming なのか？

ストリーミング処理で Iceberg に書き込む方法として、AWS では以下の選択肢があります：

- **AWS Glue Streaming**（Spark Structured Streaming ベース）
- **Amazon Managed Service for Apache Flink**
- **Amazon EMR**（Spark / Flink）

本シリーズでは Spark Structured Streaming を採用します。理由は以下の通りです：

1. **Spark エコシステムとの親和性**: 既存の Spark バッチ処理との統合が容易
2. **AWS Glue との連携**: サーバーレスでの運用が可能
3. **学習コスト**: Spark DataFrame API の延長で学べる

## 構成概要

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Kafka     │───▶│   Spark     │───▶│  Iceberg    │
│  (Source)   │    │ Structured  │    │  (MinIO)    │
│             │    │ Streaming   │    │             │
└─────────────┘    └─────────────┘    └─────────────┘
                          │
                   ┌──────┴──────┐
                   │ REST Catalog │
                   └─────────────┘
```

### 使用するコンポーネント

| コンポーネント | イメージ | 役割 |
|--------------|---------|------|
| Spark | tabulario/spark-iceberg:latest | ストリーム処理 + Jupyter Notebook |
| Iceberg REST Catalog | apache/iceberg-rest-fixture:latest | メタデータ管理 |
| MinIO | minio/minio:latest | S3 互換ストレージ |
| Kafka | apache/kafka:4.1.1 | メッセージブローカー（KRaft mode） |
| Kafka UI | provectuslabs/kafka-ui:latest | Kafka 管理 UI |

## 環境構築

本記事で使用する環境はGitHubリポジトリで公開しています。

@[card](https://github.com/toshiro3/spark-kafka-iceberg-lab)

```bash
git clone https://github.com/toshiro3/spark-kafka-iceberg-lab.git
cd spark-kafka-iceberg-lab
docker compose up -d
```

初回起動時はイメージのダウンロードと Kafka connector の取得に数分かかります。

```bash
# 起動状況の確認
docker compose ps
```

すべてのコンテナが `Up` になっていれば成功です。

### 構成ファイル

リポジトリには以下のファイルが含まれています：

| ファイル | 説明 |
|---------|------|
| `docker-compose.yml` | 全コンポーネントの定義 |
| `spark-defaults.conf` | Spark設定（Kafka connector含む） |
| `notebooks/00_environment_check.ipynb` | 動作確認用Notebook |

### ポイント：Kafka connector の追加

`tabulario/spark-iceberg` イメージには Kafka connector が含まれていません。`spark-defaults.conf` で以下の設定を追加しています：

```conf
spark.jars.packages  org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.1
```

これにより、Spark 起動時に Kafka connector が自動的にダウンロードされます。

## 動作確認

### アクセス URL

| サービス | URL |
|---------|-----|
| Jupyter Notebook | http://localhost:8888 |
| MinIO Console | http://localhost:9001 |
| Kafka UI | http://localhost:8082 |
| Spark UI | http://localhost:4040 |

### Notebook による確認

`notebooks/00_environment_check.ipynb` を開いて、各セルを実行します。

#### 1. SparkSession の確認

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.getOrCreate()

print("SparkSession 取得完了!")
print(f"Version: {spark.version}")
spark
```

#### 2. Iceberg テーブルの作成

```python
# Namespace の作成
spark.sql("CREATE NAMESPACE IF NOT EXISTS demo.test_ns")

# テーブルの作成
spark.sql("""
    CREATE TABLE IF NOT EXISTS demo.test_ns.sample_table (
        id INT,
        name STRING,
        created_at TIMESTAMP
    ) USING iceberg
""")

# データの挿入
spark.sql("""
    INSERT INTO demo.test_ns.sample_table VALUES 
        (1, 'Alice', current_timestamp()),
        (2, 'Bob', current_timestamp())
""")

spark.sql("SELECT * FROM demo.test_ns.sample_table").show()
```

#### 3. Kafka への書き込み・読み込み

```python
# Kafka にメッセージを送信
test_data = [
    ("key1", '{"id": 1, "value": "test1"}'),
    ("key2", '{"id": 2, "value": "test2"}')
]

df_to_kafka = spark.createDataFrame(test_data, ["key", "value"])

df_to_kafka.write \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("topic", "test-topic") \
    .save()
```

```python
# Kafka からメッセージを読み込み
df_from_kafka = spark.read \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("subscribe", "test-topic") \
    .option("startingOffsets", "earliest") \
    .load()

df_from_kafka.selectExpr(
    "CAST(key AS STRING)",
    "CAST(value AS STRING)",
    "topic",
    "partition",
    "offset",
    "timestamp"
).show(truncate=False)
```

#### 4. Structured Streaming のテスト

```python
# ストリーミング読み込み
streaming_df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("subscribe", "test-topic") \
    .option("startingOffsets", "earliest") \
    .load()

parsed_df = streaming_df.selectExpr(
    "CAST(key AS STRING) as key",
    "CAST(value AS STRING) as value",
    "timestamp"
)

# メモリシンクに書き込み
query = parsed_df.writeStream \
    .format("memory") \
    .queryName("test_stream") \
    .outputMode("append") \
    .start()

# 結果確認
import time
time.sleep(5)
spark.sql("SELECT * FROM test_stream").show(truncate=False)

# 停止
query.stop()
```

## トラブルシューティング

### Kafka connector が見つからない

```
AnalysisException: Failed to find data source: kafka.
```

`spark-defaults.conf` が正しくマウントされているか確認します：

```bash
docker exec spark-iceberg cat /opt/spark/conf/spark-defaults.conf | grep kafka
```

`spark.jars.packages` の行が表示されれば OK です。表示されない場合は、`docker compose down` してから再度 `docker compose up -d` を実行してください。

### MinIO に接続できない

MinIO の起動を待ってから他のサービスが起動する必要があります。`mc` コンテナが `Bucket created successfully` を出力していれば正常です：

```bash
docker compose logs mc
```

## まとめ

本記事では、Spark Structured Streaming + Kafka + Iceberg のローカル検証環境を Docker Compose で構築しました。

構築した環境で確認できたこと：
- SparkSession から Iceberg REST Catalog への接続
- Kafka へのメッセージ送受信
- Structured Streaming によるストリーム読み込み

次回は、この環境を使って Structured Streaming の基本的な使い方（Kafka からの読み込み、Iceberg への書き込み）を解説します。

## 参考リンク

- [Apache Iceberg - Spark Configuration](https://iceberg.apache.org/docs/latest/spark-configuration/)
- [Spark Structured Streaming + Kafka Integration](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html)
- [tabulario/spark-iceberg Docker Image](https://github.com/tabular-io/docker-spark-iceberg)
