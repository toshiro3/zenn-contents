---
title: "Spark Structured Streaming で Kafka から Iceberg へストリーミング書き込み"
emoji: "🌊"
type: "tech"
topics: ["spark", "kafka", "iceberg", "streaming", "python"]
published: true
---

## はじめに

この記事では、Spark Structured Streaming を使って Kafka からデータを読み込み、Apache Iceberg テーブルにリアルタイムで書き込むパイプラインを構築します。

## 前提条件

前回の記事で構築した Docker Compose 環境を使用します。

https://zenn.dev/toshiro3/articles/spark-kafka-iceberg-docker-setup

この環境には以下の依存パッケージが `spark-defaults.conf` で設定済みです：

```conf
spark.jars.packages  org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.1
```

- **spark-sql-kafka**: Kafka からの読み書きに必要
- **iceberg-spark-runtime**: `tabulario/spark-iceberg` イメージに同梱済み

## Structured Streaming とは

Spark Structured Streaming は、ストリーミングデータを DataFrame/SQL で扱える API です。

### 特徴

- **バッチと同じ API**: `spark.read` → `spark.readStream` に変えるだけ
- **マイクロバッチ処理**: 短い間隔でバッチ処理を繰り返す
- **Exactly-once 保証**: チェックポイントにより重複なし・欠損なしを実現
- **Iceberg との統合**: ストリーミングで直接 Iceberg テーブルに書き込み可能

### バッチ vs ストリーミング

```python
# バッチ読み込み
df = spark.read.format("kafka").option(...).load()

# ストリーミング読み込み
df = spark.readStream.format("kafka").option(...).load()
```

## 実装：Kafka → Iceberg パイプライン

EC サイトのユーザー行動ログをリアルタイムで処理するシナリオで実装します。

```
┌─────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Kafka     │    │ Spark Structured │    │    Iceberg      │
│   events    │───▶│    Streaming     │───▶│   user_events   │
│   topic     │    │   (JSON変換)      │    │     table       │
└─────────────┘    └──────────────────┘    └─────────────────┘
                            │
                   ┌────────┴────────┐
                   │   Checkpoint    │
                   │   (進捗管理)     │
                   └─────────────────┘
```

### データ構造

```json
{
  "event_id": "evt_0001",
  "user_id": "user_001",
  "event_type": "page_view",
  "page": "/products/123",
  "timestamp": "2024-12-31T10:00:00Z"
}
```

### Step 1: Iceberg テーブルの作成

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.getOrCreate()

# Namespace とテーブルの作成
spark.sql("CREATE NAMESPACE IF NOT EXISTS demo.streaming")

spark.sql("""
    CREATE TABLE IF NOT EXISTS demo.streaming.user_events (
        event_id STRING,
        user_id STRING,
        event_type STRING,
        page STRING,
        event_time TIMESTAMP
    ) USING iceberg
""")
```

### Step 2: テストデータを Kafka に送信

```python
import json
from datetime import datetime, timedelta

events = []
base_time = datetime.now()

for i in range(10):
    event = {
        "event_id": f"evt_{i:04d}",
        "user_id": f"user_{i % 3:03d}",
        "event_type": ["page_view", "click", "purchase"][i % 3],
        "page": f"/products/{100 + i}",
        "timestamp": (base_time + timedelta(seconds=i)).isoformat()
    }
    events.append((event["event_id"], json.dumps(event)))

df_events = spark.createDataFrame(events, ["key", "value"])

df_events.write \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("topic", "events") \
    .save()
```

### Step 3: ストリーミング処理

```python
from pyspark.sql.functions import from_json, col
from pyspark.sql.types import StructType, StructField, StringType

# JSON スキーマ定義
schema = StructType([
    StructField("event_id", StringType()),
    StructField("user_id", StringType()),
    StructField("event_type", StringType()),
    StructField("page", StringType()),
    StructField("timestamp", StringType())  # 一旦 String で受け取る
])

# Kafka からストリーミング読み込み
streaming_df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("subscribe", "events") \
    .option("startingOffsets", "earliest") \
    .load()

# JSON パースと変換
parsed_df = streaming_df \
    .select(from_json(col("value").cast("string"), schema).alias("data")) \
    .select(
        col("data.event_id"),
        col("data.user_id"),
        col("data.event_type"),
        col("data.page"),
        col("data.timestamp").cast("timestamp").alias("event_time")  # TIMESTAMP に変換
    )

# Iceberg に書き込み
query = parsed_df.writeStream \
    .format("iceberg") \
    .outputMode("append") \
    .option("checkpointLocation", "/home/iceberg/checkpoints/user_events") \
    .toTable("demo.streaming.user_events")
```

:::message
**timestamp の型変換について**

JSON の `timestamp` フィールドは ISO8601 形式の文字列（`2024-12-31T10:00:00Z`）です。

本記事では `StringType()` で受け取り、後から `.cast("timestamp")` で変換しています。これは以下の理由からです：
- JSON ソースのフォーマットが不定な場合に柔軟に対応できる
- パースエラー時のデバッグがしやすい

`TimestampType()` を直接使用することも可能ですが、タイムゾーンや日付フォーマットの不整合でパースエラーになる場合があります。
:::

### Step 4: 結果確認

```python
import time
time.sleep(5)

# データ確認
spark.sql("SELECT * FROM demo.streaming.user_events ORDER BY event_time").show()

# 件数確認
spark.sql("SELECT COUNT(*) FROM demo.streaming.user_events").show()

# クエリ停止
query.stop()
```

:::message alert
環境やデータ量によっては、書き込み完了まで 5 秒以上かかる場合があります。データが表示されない場合は、`time.sleep(10)` に変更するか、以下のように `awaitTermination` を使用してください：

```python
# 指定時間待機（タイムアウト付き）
query.awaitTermination(timeout=10)
```
:::

## チェックポイント

`checkpointLocation` はストリーミング処理の進捗を保存する場所です。

### 役割

- **オフセット管理**: Kafka のどこまで読んだかを記録
- **障害復旧**: 再起動時に続きから処理を再開
- **Exactly-once 保証**: 重複処理を防止

### 注意点

- チェックポイントは**クエリごとに一意のパス**が必要
- パスを変更すると、以前の進捗（オフセット）が引き継がれず**新規クエリとして扱われる**
- 本番環境では S3 などの永続ストレージを使用

:::message
チェックポイントのパスを変更した場合の挙動は `startingOffsets` の設定によって異なります：
- `earliest`: 最初から全データを再処理
- `latest`: 起動後に入ってきたデータのみ処理（過去データはスキップ）
:::

```python
# 例: S3 をチェックポイントに使用
.option("checkpointLocation", "s3://my-bucket/checkpoints/user_events")
```

## トリガー設定

デフォルトではできるだけ早くマイクロバッチを処理しますが、間隔を指定できます。

### processingTime

指定した間隔でバッチ処理を実行：

```python
query = parsed_df.writeStream \
    .format("iceberg") \
    .outputMode("append") \
    .option("checkpointLocation", "/home/iceberg/checkpoints/user_events") \
    .trigger(processingTime="10 seconds") \
    .toTable("demo.streaming.user_events")
```

### availableNow

現在利用可能なデータをすべて処理して停止（バッチ的な使い方）：

```python
query = parsed_df.writeStream \
    .format("iceberg") \
    .outputMode("append") \
    .option("checkpointLocation", "/home/iceberg/checkpoints/user_events") \
    .trigger(availableNow=True) \
    .toTable("demo.streaming.user_events")
```

## Iceberg スナップショット

ストリーミング書き込みでは、マイクロバッチごとにスナップショットが作成されます。

```python
spark.sql("""
    SELECT 
        snapshot_id,
        committed_at,
        operation,
        summary['added-records'] as added_records
    FROM demo.streaming.user_events.snapshots
    ORDER BY committed_at
""").show(truncate=False)
```

```
+--------------------+------------------------+---------+-------------+
|snapshot_id         |committed_at            |operation|added_records|
+--------------------+------------------------+---------+-------------+
|3039719484403124022 |2026-01-06 13:20:51.998 |append   |10           |
|8383822172186054235 |2026-01-06 13:25:13.752 |append   |1            |
|3248936071893229378 |2026-01-06 13:25:14.199 |append   |4            |
+--------------------+------------------------+---------+-------------+
```

:::message
頻繁なマイクロバッチはスナップショットを大量に生成します。本番環境では定期的な compaction が必要です。
:::

## まとめ

本記事では、Spark Structured Streaming を使って Kafka から Iceberg へのストリーミング書き込みを実装しました。

確認できたこと：
- Kafka からのストリーミング読み込み
- JSON パースと型変換
- Iceberg テーブルへのリアルタイム書き込み
- チェックポイントによる再開
- トリガー設定のオプション

次回は、ウィンドウ関数を使った集計処理（時間ウィンドウごとの集計など）を試してみようと思います。

## 参考リンク

- [Spark Structured Streaming Programming Guide](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)
- [Structured Streaming + Kafka Integration](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html)
- [Apache Iceberg - Spark Structured Streaming](https://iceberg.apache.org/docs/latest/spark-structured-streaming/)
