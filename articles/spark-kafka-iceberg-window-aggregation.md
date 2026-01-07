---
title: "Spark Structured Streaming ウィンドウ集計と Watermark の挙動"
emoji: "⏱️"
type: "tech"
topics: ["spark", "kafka", "iceberg", "streaming", "python"]
published: false
---

## はじめに

この記事では、Spark Structured Streaming のウィンドウ関数を使った時間ベースの集計処理と、Watermark（ウォーターマーク）の挙動を検証します。

前回までの記事で構築した環境を使用します。

https://zenn.dev/toshiro3/articles/spark-kafka-iceberg-docker-setup

https://zenn.dev/toshiro3/articles/spark-kafka-iceberg-streaming-basics

## ウィンドウ関数の種類

Spark Structured Streaming では、2種類のウィンドウ集計が使えます。

### Tumbling Window（固定幅ウィンドウ）

```
|-------|-------|-------|
0      30      60      90  (秒)

→ ウィンドウは重複しない
→ 各イベントは1つのウィンドウにのみ属する
```

### Sliding Window（スライディングウィンドウ）

```
|-------|
    |-------|
        |-------|
0   10  20  30  40  50  (秒)
window = 30秒, slide = 10秒

→ ウィンドウが重複する
→ 各イベントは複数のウィンドウに属する（より滑らかな集計）
```

## 実装：Tumbling Window

イベントタイプ別に30秒ウィンドウで集計します。

### テーブル作成

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.getOrCreate()

spark.sql("CREATE NAMESPACE IF NOT EXISTS demo.streaming")

spark.sql("""
    CREATE TABLE IF NOT EXISTS demo.streaming.event_counts (
        window_start TIMESTAMP,
        window_end TIMESTAMP,
        event_type STRING,
        count BIGINT
    ) USING iceberg
""")
```

### ストリーミング処理

```python
from pyspark.sql.functions import from_json, col, window
from pyspark.sql.types import StructType, StructField, StringType

schema = StructType([
    StructField("event_id", StringType()),
    StructField("user_id", StringType()),
    StructField("event_type", StringType()),
    StructField("page", StringType()),
    StructField("timestamp", StringType())
])

# Kafka からストリーミング読み込み
streaming_df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("subscribe", "events") \
    .option("startingOffsets", "latest") \
    .load()

# JSON パース
parsed_df = streaming_df \
    .select(from_json(col("value").cast("string"), schema).alias("data")) \
    .select(
        col("data.event_id"),
        col("data.user_id"),
        col("data.event_type"),
        col("data.page"),
        col("data.timestamp").cast("timestamp").alias("event_time")
    )

# Watermark + Tumbling Window（30秒ウィンドウ）で集計
windowed_counts = parsed_df \
    .withWatermark("event_time", "10 seconds") \
    .groupBy(
        window(col("event_time"), "30 seconds"),
        col("event_type")
    ) \
    .count()

# 結果を整形
result_df = windowed_counts.select(
    col("window.start").alias("window_start"),
    col("window.end").alias("window_end"),
    col("event_type"),
    col("count")
)

# コンソールに出力（デバッグ用）
query = result_df.writeStream \
    .format("console") \
    .outputMode("update") \
    .trigger(processingTime="10 seconds") \
    .start()
```

### 実行結果例

20件のイベントを1秒間隔で送信し、Iceberg に `outputMode("append")` で書き込んだ結果：

```python
spark.sql("SELECT * FROM demo.streaming.event_counts ORDER BY window_start").show()
```

```
+-------------------+-------------------+----------+-----+
|window_start       |window_end         |event_type|count|
+-------------------+-------------------+----------+-----+
|2026-01-06 22:50:30|2026-01-06 22:51:00|click     |5    |
|2026-01-06 22:50:30|2026-01-06 22:51:00|page_view |2    |
|2026-01-06 22:50:30|2026-01-06 22:51:00|purchase  |5    |
+-------------------+-------------------+----------+-----+
```

集計されたイベントは **12件**（5+2+5）です。20件送信したのに、8件足りません。

## Watermark と未確定ウィンドウ

### なぜ 8件が集計されないのか？

残りの 8件は、まだ**確定していないウィンドウ**に含まれています。

### Watermark の役割

Watermark は「これより古いイベントは来ない」という境界を表します。

```python
.withWatermark("event_time", "10 seconds")
```

この設定では、現在処理中のイベント時刻から10秒前までの遅延データを許容します。

Watermark には2つの重要な役割があります：

1. **ウィンドウの確定**: Watermark がウィンドウ終了時刻を超えると、そのウィンドウは「確定」し、結果が出力される
2. **メモリの解放**: 確定したウィンドウの集計情報は Spark の内部メモリ（State）から削除される

:::message
Watermark を設定しないと、すべてのウィンドウの状態がメモリに保持され続け、長時間運用するとメモリ不足になります。実運用では必ず設定してください。
:::

### ウィンドウの確定条件

```
Watermark = 最新イベント時刻 - 10秒

確定条件: Watermark > ウィンドウ終了時刻
```

今回のケースでは：

- **Window 1（22:50:30 - 22:51:00）**: 12件が入り、Watermark がウィンドウ終了時刻を超えたので確定・出力
- **Window 2（22:51:00 - 22:51:30）**: 残り8件が入ったが、Watermark が終了時刻を超えていないため未確定

### 検証：追加イベントで未確定ウィンドウを確定させる

追加で10件のイベントを送信して Watermark を進めます。

```python
# 追加で10件送信
send_events_to_kafka(num_events=10, interval=1)

# 40秒待機（Watermark + ウィンドウ幅）
import time
time.sleep(40)

# 結果確認
spark.sql("SELECT * FROM demo.streaming.event_counts ORDER BY window_start").show()
```

```
+-------------------+-------------------+----------+-----+
|window_start       |window_end         |event_type|count|
+-------------------+-------------------+----------+-----+
|2026-01-06 22:50:30|2026-01-06 22:51:00|click     |5    |
|2026-01-06 22:50:30|2026-01-06 22:51:00|page_view |2    |
|2026-01-06 22:50:30|2026-01-06 22:51:00|purchase  |5    |
|2026-01-06 22:51:00|2026-01-06 22:51:30|click     |5    |  ← 追加
|2026-01-06 22:51:00|2026-01-06 22:51:30|page_view |2    |  ← 追加
|2026-01-06 22:51:00|2026-01-06 22:51:30|purchase  |1    |  ← 追加
|...                |...                |...       |...  |
+-------------------+-------------------+----------+-----+
```

未確定だった Window 2（22:51:00 - 22:51:30）の 8件（5+2+1）が確定して出力されました。

### outputMode による挙動の違い

| モード | 出力タイミング | 特徴 | 活用シーン |
|--------|--------------|------|-----------|
| `update` | 更新があるたびに出力 | 速報値。後から値が変わる可能性あり | リアルタイム監視、ダッシュボード |
| `append` | Watermark がウィンドウ終了時刻を超えた後 | 確定値のみ出力。重複なし | 課金、監査、データ蓄積 |
| `complete` | 全ウィンドウを毎回出力 | 全量出力 | 小規模データの全量表示 |

:::message
Iceberg への書き込みでは `append` と `complete` モードがサポートされています。`update` モードは非サポートのため、コンソールやメモリシンクでのデバッグ用途に限られます。
:::

## 実装：Sliding Window

```python
# Sliding Window: 30秒ウィンドウを10秒ごとにスライド
sliding_counts = parsed_df \
    .withWatermark("event_time", "10 seconds") \
    .groupBy(
        window(col("event_time"), "30 seconds", "10 seconds"),  # windowDuration, slideDuration
        col("event_type")
    ) \
    .count()
```

### 実行結果例

15件のイベントを Sliding Window（30秒ウィンドウ、10秒スライド）で集計した場合（`outputMode("update")`）：

```
-------------------------------------------
Batch: 3
-------------------------------------------
+-------------------+-------------------+----------+-----+
|window_start       |window_end         |event_type|count|
+-------------------+-------------------+----------+-----+
|2026-01-06 23:49:20|2026-01-06 23:49:50|page_view |3    |
|2026-01-06 23:49:30|2026-01-06 23:50:00|page_view |5    |
|2026-01-06 23:49:30|2026-01-06 23:50:00|purchase  |4    |
|2026-01-06 23:49:40|2026-01-06 23:50:10|page_view |4    |
|2026-01-06 23:49:40|2026-01-06 23:50:10|click     |5    |
|2026-01-06 23:49:30|2026-01-06 23:50:00|click     |6    |
+-------------------+-------------------+----------+-----+
```

ウィンドウ開始時刻が10秒ずつずれています（23:49:20、23:49:30、23:49:40...）。1つのイベントが複数のウィンドウにカウントされるため、各ウィンドウの count の合計は送信したイベント数（15件）を超えます。

## バッチ処理での集計結果検証

ストリーミングの集計結果が正しいか、Kafka から生データを読み込んで検証できます。

```python
# Kafka から全データをバッチで読み込み
raw_df = spark.read \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("subscribe", "events") \
    .option("startingOffsets", "earliest") \
    .option("endingOffsets", "latest") \
    .load()

# JSON パース
events_df = raw_df \
    .select(from_json(col("value").cast("string"), schema).alias("data")) \
    .select(
        col("data.event_id"),
        col("data.event_type"),
        col("data.timestamp").cast("timestamp").alias("event_time")
    )

# 特定のウィンドウ範囲でフィルタして件数を確認
filtered_df = events_df.filter(
    (col("event_time") >= "2026-01-06 23:49:30") & 
    (col("event_time") < "2026-01-06 23:50:00")
)

filtered_df.groupBy("event_type").count().show()
```

Sliding Window の集計結果（click=6, purchase=4, page_view=5）と、このバッチ処理の結果が一致することで、ストリーミング集計が正しく動作していることを確認できます。

## ストリーミング処理の特性

### バッチごとの集計について

`outputMode("update")` では、各バッチで「その時点で読み込めたイベント」のみが集計されます。

```
Batch 1: Kafka からイベント数件を読み込み → 該当ウィンドウを集計・出力
Batch 2: 追加イベントを読み込み → ウィンドウの count を更新・出力
Batch 3: さらに追加イベントを読み込み → ウィンドウの count を更新・出力
```

`update` モードの出力は速報値であり、後のバッチで更新される可能性があります。

### 速報値 vs 確定値

| 状態 | 出力モード | 説明 | メモリ状態 |
|------|-----------|------|-----------|
| 速報値 | `update` | 到着したデータから順次集計。後から値が変わる可能性あり | ウィンドウの状態を保持し続ける |
| 確定値 | `append` | Watermark によりデータ到着終了が保証された後に出力 | 出力後にメモリから解放 |

## 大量バックログがある場合の考慮

Kafka に大量のメッセージが蓄積されている状態で `startingOffsets="earliest"` で開始すると、直近のイベントが集計対象になるまで時間がかかります。

### 対処法

| 方法 | 説明 |
|------|------|
| `startingOffsets="latest"` | 起動後のイベントのみ処理 |
| タイムスタンプ指定 | 特定時刻以降から開始 |
| バッチ + ストリーム併用 | 過去はバッチ処理、以降はストリーム |

```python
# 1時間前から開始する例
import json
from datetime import datetime, timedelta

start_time = int((datetime.now() - timedelta(hours=1)).timestamp() * 1000)

streaming_df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("subscribe", "events") \
    .option("startingOffsetsByTimestamp", 
            json.dumps({"events": {0: start_time}})) \
    .load()
```

## まとめ

本記事では、Spark Structured Streaming のウィンドウ集計と Watermark の挙動を検証しました。

確認できたこと：
- Tumbling Window と Sliding Window の違い
- Watermark による遅延データの許容と確定条件
- outputMode（update / append）による出力タイミングの違い
- 速報値と確定値の概念
- バッチ処理による集計結果の検証

次回は、AWS Glue Streaming での実行を試してみます。

## 参考リンク

- [Spark Structured Streaming - Window Operations](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#window-operations-on-event-time)
- [Handling Late Data and Watermarking](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#handling-late-data-and-watermarking)
- [Apache Iceberg - Spark Structured Streaming](https://iceberg.apache.org/docs/latest/spark-structured-streaming/)
