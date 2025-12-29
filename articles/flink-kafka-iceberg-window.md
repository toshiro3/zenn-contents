---
title: "Flink SQL ウィンドウ関数の使い分けとアイドルタイムアウト"
emoji: "⏱️"
type: "tech"
topics: ["flink", "kafka", "iceberg", "streaming", "docker"]
published: true
---

## はじめに

前回の記事では、Flink SQLを使ってKafkaからIcebergへのストリーミング書き込みと、TUMBLEウィンドウによる集計を検証しました。

https://zenn.dev/toshiro3/articles/flink-kafka-iceberg-streaming

本記事では、さらに以下のトピックを検証します。

- スライディングウィンドウ（HOP）
- セッションウィンドウ（SESSION）
- アイドルタイムアウトの設定と制限

検証環境は前回と同じ`flink-kafka-iceberg-lab`を使用します。

https://github.com/toshiro3/flink-kafka-iceberg-lab

## ウィンドウ関数の種類

Flink SQLでは3種類のウィンドウ関数が用意されています。

| ウィンドウ | 特徴 | ユースケース |
|-----------|------|-------------|
| TUMBLE | 固定長、重複なし | 1分ごとの集計など |
| HOP | 固定長、重複あり | 移動平均、トレンド分析 |
| SESSION | 可変長、アクティビティベース | ユーザー行動分析 |

```
TUMBLE（前回検証済み）: 重複なし
|  Window 1  |  Window 2  |  Window 3  |
|   1分間    |   1分間    |   1分間    |

HOP: 重複あり（2分ウィンドウを1分ごとにスライド）
|     Window 1 (2分)     |
      |     Window 2 (2分)     |
            |     Window 3 (2分)     |
|--1分--|--1分--|--1分--|--1分--|

SESSION: アクティビティベース（ギャップで区切る）
|--イベント--イベント--|  gap  |--イベント--イベント--イベント--|
|     Session 1       |       |         Session 2            |
```

## 環境準備

### 環境の起動

```bash
cd flink-kafka-iceberg-lab
docker compose up -d
```

Kafkaトピックは、プロデューサーからデータを送信する際に自動作成されるため、事前作成は不要です。

### ターミナルの使い分け

本記事では3つのターミナルを使い分けます。

| ターミナル | 用途 | 起動コマンド |
|-----------|------|-------------|
| **ターミナル1（Flink ストリーミング）** | ジョブの作成・実行 | `docker compose run --rm sql-client` |
| **ターミナル2（Kafka）** | トピックへのデータ送信 | `docker compose exec kafka ...` |
| **ターミナル3（Flink バッチ）** | 結果の確認 | `docker compose run --rm sql-client` |

以降の手順では、どのターミナルで操作するかを明示します。

### カタログの使い分け（前回の復習）

前回の記事で説明した通り、Flink SQL Clientでは2つのカタログを使い分けます。

| カタログ | 種類 | 永続化 | 用途 |
|---------|------|--------|------|
| default_catalog | インメモリ | ❌ | Kafkaなどのコネクタテーブル |
| iceberg_catalog | REST Catalog | ✅ | Icebergテーブル（データレイク） |

**なぜ分けるのか？**

- KafkaテーブルはWATERMARK定義が必要だが、IcebergカタログのメタデータストアにはWATERMARK定義を永続化できないため、分離して管理する
- `default_catalog`（インメモリ）をKafka接続定義の置き場にすることで、**「消えても良い一時的な定義（Kafka）」と「永続化されるデータ資産（Iceberg）」**を論理的に整理できる

## スライディングウィンドウ（HOP）

HOPウィンドウは、固定長のウィンドウをスライドさせながら集計します。同じイベントが複数のウィンドウに含まれるのが特徴です。

### テーブルの作成

**【ターミナル1: Flink ストリーミング】**

Flink SQL Clientを起動します。

```bash
docker compose run --rm sql-client
```

まず、Icebergカタログに集計結果を保存するテーブルを作成します。

```sql
-- Icebergカタログでテーブル作成
USE CATALOG iceberg_catalog;
CREATE DATABASE IF NOT EXISTS demo;
USE demo;

-- チェックポイント間隔を設定（Icebergへの書き込みに必要）
SET 'execution.checkpointing.interval' = '10s';

-- HOPウィンドウ用の集計テーブル
CREATE TABLE IF NOT EXISTS event_counts_hop (
    window_start TIMESTAMP(6),
    window_end TIMESTAMP(6),
    event_type STRING,
    event_count BIGINT
);
```

次に、default_catalogにKafkaソーステーブルを作成します。

```sql
-- default_catalogでKafkaソーステーブルを作成
USE CATALOG default_catalog;

CREATE TABLE kafka_hop_events (
    event_id STRING,
    user_id STRING,
    event_type STRING,
    event_time TIMESTAMP(3),
    WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'hop-events',
    'properties.bootstrap.servers' = 'kafka:29092',
    'properties.group.id' = 'flink-hop-consumer',
    'scan.startup.mode' = 'earliest-offset',
    'format' = 'json',
    'json.timestamp-format.standard' = 'ISO-8601'
);
```

:::message
`json.timestamp-format.standard` を `ISO-8601` に設定することで、`2025-12-29T12:00:10` のような形式のタイムスタンプを正しくパースできます。
:::

### ジョブの実行

**【ターミナル1: Flink ストリーミング（続き）】**

```sql
-- HOPウィンドウ集計ジョブ
-- 2分間のウィンドウを1分ごとにスライド
INSERT INTO iceberg_catalog.demo.event_counts_hop
SELECT 
    window_start,
    window_end,
    event_type,
    COUNT(*) as event_count
FROM TABLE(
    HOP(TABLE kafka_hop_events, DESCRIPTOR(event_time), INTERVAL '1' MINUTE, INTERVAL '2' MINUTE)
)
GROUP BY window_start, window_end, event_type;
```

HOP関数のパラメータは以下の通りです。

| パラメータ | 値 | 説明 |
|-----------|-----|------|
| 第1引数 | TABLE kafka_hop_events | ソーステーブル |
| 第2引数 | DESCRIPTOR(event_time) | 時間属性カラム |
| 第3引数 | INTERVAL '1' MINUTE | スライド間隔 |
| 第4引数 | INTERVAL '2' MINUTE | ウィンドウサイズ |

### テストデータの送信

**【ターミナル2: Kafka】**

Kafkaプロデューサーを起動します。

```bash
docker compose exec kafka /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic hop-events
```

以下のデータを送信します（2分間にまたがるデータ）。

```json
{"event_id":"h001","user_id":"u001","event_type":"click","event_time":"2025-12-29T12:00:10"}
{"event_id":"h002","user_id":"u002","event_type":"click","event_time":"2025-12-29T12:00:30"}
{"event_id":"h003","user_id":"u001","event_type":"view","event_time":"2025-12-29T12:01:10"}
{"event_id":"h004","user_id":"u003","event_type":"click","event_time":"2025-12-29T12:01:40"}
{"event_id":"h005","user_id":"u002","event_type":"purchase","event_time":"2025-12-29T12:02:20"}
```

ウィンドウを確定させるために、後の時刻のデータも送信します。

```json
{"event_id":"h006","user_id":"u001","event_type":"click","event_time":"2025-12-29T12:04:00"}
```

`Ctrl+C`で終了します。

### 結果の確認

**【ターミナル3: Flink バッチ】**

別のSQL Clientを起動して結果を確認します。

```bash
docker compose run --rm sql-client
```

```sql
USE CATALOG iceberg_catalog;
USE demo;

SET 'execution.runtime-mode' = 'batch';
SELECT * FROM event_counts_hop ORDER BY window_start, event_type;
```

結果例：

| window_start | window_end | event_type | event_count |
|--------------|------------|------------|-------------|
| 11:59:00 | 12:01:00 | click | 2 |
| 12:00:00 | 12:02:00 | click | 3 |
| 12:00:00 | 12:02:00 | view | 1 |
| 12:01:00 | 12:03:00 | click | 1 |
| 12:01:00 | 12:03:00 | purchase | 1 |
| 12:01:00 | 12:03:00 | view | 1 |

HOPウィンドウでは、同じイベントが複数のウィンドウに含まれます。例えば、12:01:10のviewイベントは「12:00〜12:02」と「12:01〜12:03」の両方のウィンドウに含まれています。

## セッションウィンドウ（SESSION）

セッションウィンドウは、アクティビティの「ギャップ」に基づいてウィンドウを動的に作成します。ユーザーごとの行動分析に適しています。

### テーブルの作成

**【ターミナル1: Flink ストリーミング】**

```sql
-- Icebergカタログでテーブル作成
USE CATALOG iceberg_catalog;
CREATE DATABASE IF NOT EXISTS demo;
USE demo;

SET 'execution.checkpointing.interval' = '10s';

-- セッションウィンドウ用の集計テーブル
CREATE TABLE IF NOT EXISTS user_sessions (
    user_id STRING,
    session_start TIMESTAMP(6),
    session_end TIMESTAMP(6),
    event_count BIGINT
);

-- default_catalogでKafkaソーステーブルを作成
USE CATALOG default_catalog;

CREATE TABLE kafka_session_events (
    event_id STRING,
    user_id STRING,
    event_type STRING,
    event_time TIMESTAMP(3),
    WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'session-events',
    'properties.bootstrap.servers' = 'kafka:29092',
    'properties.group.id' = 'flink-session-consumer',
    'scan.startup.mode' = 'earliest-offset',
    'format' = 'json',
    'json.timestamp-format.standard' = 'ISO-8601'
);
```

### ジョブの実行

**【ターミナル1: Flink ストリーミング（続き）】**

```sql
-- セッションウィンドウ集計ジョブ（30秒のギャップでセッション区切り）
INSERT INTO iceberg_catalog.demo.user_sessions
SELECT 
    user_id,
    window_start,
    window_end,
    COUNT(*) as event_count
FROM TABLE(
    SESSION(TABLE kafka_session_events PARTITION BY user_id, DESCRIPTOR(event_time), INTERVAL '30' SECOND)
)
GROUP BY user_id, window_start, window_end;
```

SESSION関数のパラメータは以下の通りです。

| パラメータ | 値 | 説明 |
|-----------|-----|------|
| PARTITION BY | user_id | セッションを分けるキー |
| DESCRIPTOR | event_time | 時間属性カラム |
| INTERVAL | 30 SECOND | セッションギャップ（この時間イベントがないと新しいセッション） |

### テストデータの送信

**【ターミナル2: Kafka】**

```bash
docker compose exec kafka /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic session-events
```

以下のデータを**event_time順**に送信します。

```json
{"event_id":"s001","user_id":"u001","event_type":"click","event_time":"2025-12-29T13:00:00"}
{"event_id":"s006","user_id":"u002","event_type":"click","event_time":"2025-12-29T13:00:10"}
{"event_id":"s002","user_id":"u001","event_type":"view","event_time":"2025-12-29T13:00:15"}
{"event_id":"s007","user_id":"u002","event_type":"view","event_time":"2025-12-29T13:00:20"}
{"event_id":"s003","user_id":"u001","event_type":"click","event_time":"2025-12-29T13:00:25"}
{"event_id":"s004","user_id":"u001","event_type":"view","event_time":"2025-12-29T13:01:30"}
{"event_id":"s005","user_id":"u001","event_type":"click","event_time":"2025-12-29T13:01:45"}
```

:::message alert
データは**event_time順**に送信することが重要です。ユーザーごとにまとめて送信すると、ウォーターマークの進行によって後から送信したユーザーのデータが「遅延データ」として破棄される可能性があります。この問題については後述の「アイドルタイムアウト」で詳しく解説します。
:::

ウィンドウを確定させるために、後の時刻のデータも送信します。

```json
{"event_id":"s008","user_id":"u001","event_type":"click","event_time":"2025-12-29T13:05:00"}
{"event_id":"s009","user_id":"u002","event_type":"click","event_time":"2025-12-29T13:05:00"}
```

### 結果の確認

**【ターミナル3: Flink バッチ】**

```sql
USE CATALOG iceberg_catalog;
USE demo;

SET 'execution.runtime-mode' = 'batch';
SELECT * FROM user_sessions ORDER BY user_id, session_start;
```

結果例：

| user_id | session_start | session_end | event_count |
|---------|---------------|-------------|-------------|
| u001 | 13:00:00 | 13:00:55 | 3 |
| u001 | 13:01:30 | 13:02:15 | 2 |
| u002 | 13:00:10 | 13:00:50 | 2 |

u001は2つのセッションに分かれています。13:00:25から13:01:30まで65秒のギャップがあるため、別のセッションとして認識されています。

`session_end`は「最後のイベント時刻 + ギャップ時間（30秒）」になっています。

## アイドルタイムアウトの設定と制限

### 問題の背景

セッションウィンドウの検証で触れたように、ウォーターマークの進行によって「遅延データ」が破棄される問題があります。これはKafkaの複数パーティションを使用する場合に特に顕著になります。

```
パーティション0: イベントが継続的に流れる → ウォーターマークが進む
パーティション1: イベントが来ない（アイドル状態）→ ウォーターマークが進まない

全体のウォーターマーク = min(パーティション0, パーティション1)
```

Flinkは全パーティションの最小ウォーターマークを採用するため、アイドルパーティションがあると全体のウォーターマークが進まず、ウィンドウが確定しません。

### 問題の再現

アイドルパーティション問題を再現するために、2パーティションのトピックを作成します。

パーティション数を指定する必要があるため、事前にトピックを作成します。

**【ターミナル2: Kafka】**

```bash
# アイドルタイムアウトなしの検証用
docker compose exec kafka /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --topic multi-partition-events --partitions 2 --replication-factor 1

# アイドルタイムアウトありの検証用
docker compose exec kafka /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --topic multi-partition-events-idle --partitions 2 --replication-factor 1
```

#### アイドルタイムアウトなしでの検証

**【ターミナル1: Flink ストリーミング】**

```sql
-- Icebergカタログでテーブル作成
USE CATALOG iceberg_catalog;
CREATE DATABASE IF NOT EXISTS demo;
USE demo;

SET 'execution.checkpointing.interval' = '10s';

CREATE TABLE IF NOT EXISTS partition_test (
    window_start TIMESTAMP(6),
    window_end TIMESTAMP(6),
    event_type STRING,
    event_count BIGINT
);

-- default_catalogでアイドルタイムアウトなしのKafkaソーステーブルを作成
USE CATALOG default_catalog;

CREATE TABLE kafka_multi_partition (
    event_id STRING,
    user_id STRING,
    event_type STRING,
    event_time TIMESTAMP(3),
    WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'multi-partition-events',
    'properties.bootstrap.servers' = 'kafka:29092',
    'properties.group.id' = 'flink-multi-partition-consumer',
    'scan.startup.mode' = 'earliest-offset',
    'format' = 'json',
    'json.timestamp-format.standard' = 'ISO-8601'
);

-- TUMBLEウィンドウ集計
INSERT INTO iceberg_catalog.demo.partition_test
SELECT 
    window_start,
    window_end,
    event_type,
    COUNT(*) as event_count
FROM TABLE(
    TUMBLE(TABLE kafka_multi_partition, DESCRIPTOR(event_time), INTERVAL '1' MINUTE)
)
GROUP BY window_start, window_end, event_type;
```

**【ターミナル2: Kafka】**

パーティション0にのみデータを送信します。

```bash
docker compose exec kafka /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic multi-partition-events \
  --property parse.key=true \
  --property key.separator=:
```

```
key0:{"event_id":"m001","user_id":"u001","event_type":"click","event_time":"2025-12-29T14:00:10"}
key0:{"event_id":"m002","user_id":"u001","event_type":"view","event_time":"2025-12-29T14:00:30"}
key0:{"event_id":"m003","user_id":"u001","event_type":"click","event_time":"2025-12-29T14:00:50"}
key0:{"event_id":"m004","user_id":"u001","event_type":"click","event_time":"2025-12-29T14:02:00"}
```

**【ターミナル3: Flink バッチ】**

結果を確認しても、データは表示されません。

```sql
USE CATALOG iceberg_catalog;
USE demo;
SET 'execution.runtime-mode' = 'batch';
SELECT * FROM partition_test ORDER BY window_start, event_type;
-- 結果なし
```

```
パーティション0: ウォーターマーク 14:01:55 (14:02:00 - 5秒)
パーティション1: ウォーターマーク -∞（データなし）
全体 = min(14:01:55, -∞) = -∞ → ウィンドウが確定しない
```

### アイドルタイムアウトによる解決

`scan.watermark.idle-timeout`オプションを設定することで、アイドルパーティションをウォーターマーク計算から除外できます。

まず、既存のジョブをFlink Web UI（http://localhost:8081）から停止します。

別のトピックとテーブルを使用して、アイドルタイムアウトありの検証を行います。

**【ターミナル1: Flink ストリーミング】**

```sql
-- Icebergカタログでテーブル作成
USE CATALOG iceberg_catalog;
USE demo;

SET 'execution.checkpointing.interval' = '10s';

CREATE TABLE IF NOT EXISTS partition_test_idle (
    window_start TIMESTAMP(6),
    window_end TIMESTAMP(6),
    event_type STRING,
    event_count BIGINT
);

-- default_catalogでアイドルタイムアウトを設定したKafkaソーステーブルを作成
USE CATALOG default_catalog;

CREATE TABLE kafka_multi_partition_idle (
    event_id STRING,
    user_id STRING,
    event_type STRING,
    event_time TIMESTAMP(3),
    WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'multi-partition-events-idle',
    'properties.bootstrap.servers' = 'kafka:29092',
    'properties.group.id' = 'flink-idle-timeout-consumer',
    'scan.startup.mode' = 'earliest-offset',
    'format' = 'json',
    'json.timestamp-format.standard' = 'ISO-8601',
    'scan.watermark.idle-timeout' = '5s'
);

-- TUMBLEウィンドウ集計
INSERT INTO iceberg_catalog.demo.partition_test_idle
SELECT 
    window_start,
    window_end,
    event_type,
    COUNT(*) as event_count
FROM TABLE(
    TUMBLE(TABLE kafka_multi_partition_idle, DESCRIPTOR(event_time), INTERVAL '1' MINUTE)
)
GROUP BY window_start, window_end, event_type;
```

**【ターミナル2: Kafka】**

パーティション0にのみデータを送信します。

```bash
docker compose exec kafka /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic multi-partition-events-idle \
  --property parse.key=true \
  --property key.separator=:
```

```
key0:{"event_id":"m011","user_id":"u001","event_type":"click","event_time":"2025-12-29T16:00:10"}
key0:{"event_id":"m012","user_id":"u001","event_type":"view","event_time":"2025-12-29T16:00:30"}
key0:{"event_id":"m013","user_id":"u001","event_type":"click","event_time":"2025-12-29T16:00:50"}
key0:{"event_id":"m014","user_id":"u001","event_type":"click","event_time":"2025-12-29T16:02:00"}
```

**【ターミナル3: Flink バッチ】**

20〜30秒待ってから結果を確認すると、今度はデータが表示されます。

```sql
USE CATALOG iceberg_catalog;
USE demo;
SET 'execution.runtime-mode' = 'batch';
SELECT * FROM partition_test_idle ORDER BY window_start, event_type;
```

| window_start | window_end | event_type | event_count |
|--------------|------------|------------|-------------|
| 16:00:00 | 16:01:00 | click | 2 |
| 16:00:00 | 16:01:00 | view | 1 |

```
アイドルタイムアウト5秒:
パーティション0: ウォーターマーク 16:01:55
パーティション1: 5秒間データなし → アイドルとしてマーク（無視される）
全体 = 16:01:55 → ウィンドウ確定する
```

### アイドルタイムアウトの注意点

| 項目 | 内容 |
|------|------|
| メリット | アイドルパーティションがあってもウィンドウが進む |
| デメリット | アイドル後に遅れて来たデータは「遅延データ」として破棄される可能性 |
| 推奨値 | データの到着間隔より少し長めに設定（例：通常10秒間隔なら15〜30秒） |

アイドルタイムアウトを短く設定しすぎると、一時的にデータが途切れただけでパーティションがアイドル扱いになり、その後に来たデータが破棄されるリスクがあります。

## まとめ

### ウィンドウ関数の使い分け

| ウィンドウ | 特徴 | ユースケース |
|-----------|------|-------------|
| TUMBLE | 固定長、重複なし | 1分/1時間ごとの定期集計 |
| HOP | 固定長、重複あり | 移動平均、直近N分のトレンド |
| SESSION | 可変長、ギャップベース | ユーザーセッション分析 |

### 重要なポイント

1. **チェックポイント設定**: `SET 'execution.checkpointing.interval' = '10s';` がないとIcebergへの書き込みが行われない

2. **タイムスタンプフォーマット**: ISO-8601形式を使う場合は `'json.timestamp-format.standard' = 'ISO-8601'` を設定

3. **データ送信順序**: セッションウィンドウなどでは、event_time順にデータを送信しないとウォーターマークの問題で遅延データが破棄される

4. **アイドルタイムアウト**: 複数パーティションで偏りがある場合は `'scan.watermark.idle-timeout'` を設定

5. **カタログの使い分け**: Kafkaソーステーブルは`default_catalog`、Icebergテーブルは`iceberg_catalog`に作成

### 次回予告

次回は以下のトピックを検証予定です。

- 複数パーティションでの並列処理
- Savepointからのジョブ再開

## 参考資料

- [Apache Flink公式ドキュメント - Window TVF](https://nightlies.apache.org/flink/flink-docs-release-1.18/docs/dev/table/sql/queries/window-tvf/)
- [Flink Kafka Connector](https://nightlies.apache.org/flink/flink-docs-release-1.18/docs/connectors/table/kafka/)
- [Apache Iceberg - Flink](https://iceberg.apache.org/docs/latest/flink/)
