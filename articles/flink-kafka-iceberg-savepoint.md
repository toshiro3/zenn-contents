---
title: "Flink SQL Savepointからのジョブ再開"
emoji: "💾"
type: "tech"
topics: ["flink", "kafka", "iceberg", "streaming", "docker"]
published: true
---

## はじめに

前回の記事では、Flink SQLのウィンドウ関数（HOP、SESSION）とアイドルタイムアウトの設定について検証しました。

https://zenn.dev/toshiro3/articles/flink-kafka-iceberg-window

本記事では、Savepointを使ったジョブの停止と再開について検証します。

検証環境は前回と同じ`flink-kafka-iceberg-lab`を使用します。

https://github.com/toshiro3/flink-kafka-iceberg-lab

## Savepointとは

Savepointは、Flinkジョブの状態を特定時点でスナップショットとして保存する機能です。保存した状態から後でジョブを再開できます。

### Checkpointとの違い

| 項目 | Checkpoint | Savepoint |
|------|------------|-----------|
| 目的 | 障害からの自動復旧 | 計画的な停止・再開 |
| トリガー | 自動（定期的） | 手動 |
| 保存先 | 設定されたストレージ | 指定したパス |
| 用途 | 障害復旧 | ジョブ更新、クラスタ移行 |

### ユースケース

| シナリオ | 説明 |
|---------|------|
| ジョブの更新 | SQLやロジックを変更して再デプロイ |
| 計画メンテナンス | クラスタのメンテナンス時に状態を保存 |
| クラスタ移行 | 別のFlinkクラスタにジョブを移動 |

## 環境準備

### 環境の起動

```bash
cd flink-kafka-iceberg-lab
docker compose up -d
```

### ターミナルの使い分け

本記事では3つのターミナルを使い分けます。

| ターミナル | 用途 | 起動コマンド |
|-----------|------|-------------|
| **ターミナル1（Flink ストリーミング）** | ジョブの作成・実行 | `docker compose run --rm sql-client` |
| **ターミナル2（Kafka / Flink CLI）** | データ送信、Savepoint操作 | `docker compose exec ...` |
| **ターミナル3（Flink バッチ）** | 結果の確認 | `docker compose run --rm sql-client` |

## Savepoint検証

### 検証の流れ

1. テスト用のジョブを作成・実行
2. データを送信して集計結果を確認
3. Savepointを取得してジョブを停止
4. 停止中にKafkaにデータを送信
5. Savepointからジョブを再開
6. 停止中のデータが正しく処理されたか確認

### ステップ1: テスト用ジョブの作成

**【ターミナル1: Flink ストリーミング】**

```bash
docker compose run --rm sql-client
```

```sql
-- Icebergカタログでテーブル作成
USE CATALOG iceberg_catalog;
CREATE DATABASE IF NOT EXISTS demo;
USE demo;

SET 'execution.checkpointing.interval' = '10s';

CREATE TABLE IF NOT EXISTS savepoint_test (
    window_start TIMESTAMP(6),
    window_end TIMESTAMP(6),
    event_type STRING,
    event_count BIGINT
);

-- default_catalogでKafkaソーステーブルを作成
USE CATALOG default_catalog;

CREATE TABLE kafka_savepoint_events (
    event_id STRING,
    user_id STRING,
    event_type STRING,
    event_time TIMESTAMP(3),
    WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'savepoint-events',
    'properties.bootstrap.servers' = 'kafka:29092',
    'properties.group.id' = 'flink-savepoint-consumer',
    'scan.startup.mode' = 'earliest-offset',
    'format' = 'json',
    'json.timestamp-format.standard' = 'ISO-8601'
);

-- TUMBLEウィンドウ集計ジョブ
INSERT INTO iceberg_catalog.demo.savepoint_test
SELECT 
    window_start,
    window_end,
    event_type,
    COUNT(*) as event_count
FROM TABLE(
    TUMBLE(TABLE kafka_savepoint_events, DESCRIPTOR(event_time), INTERVAL '1' MINUTE)
)
GROUP BY window_start, window_end, event_type;
```

ジョブが実行されたら、Flink Web UI（http://localhost:8081）でジョブIDを確認します。

### ステップ2: データ送信と結果確認

**【ターミナル2: Kafka】**

```bash
docker compose exec kafka /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic savepoint-events
```

以下のデータを送信します。

```json
{"event_id":"sp001","user_id":"u001","event_type":"click","event_time":"2025-12-30T10:00:10"}
{"event_id":"sp002","user_id":"u001","event_type":"view","event_time":"2025-12-30T10:00:30"}
{"event_id":"sp003","user_id":"u002","event_type":"click","event_time":"2025-12-30T10:00:50"}
```

ウィンドウを確定させるために、後の時刻のデータも送信します。

```json
{"event_id":"sp004","user_id":"u001","event_type":"click","event_time":"2025-12-30T10:02:00"}
```

`Ctrl+C`で終了します。

**【ターミナル3: Flink バッチ】**

```bash
docker compose run --rm sql-client
```

```sql
USE CATALOG iceberg_catalog;
USE demo;

SET 'execution.runtime-mode' = 'batch';
SELECT * FROM savepoint_test ORDER BY window_start, event_type;
```

結果例：

| window_start | window_end | event_type | event_count |
|--------------|------------|------------|-------------|
| 10:00:00 | 10:01:00 | click | 2 |
| 10:00:00 | 10:01:00 | view | 1 |

### ステップ3: Savepointの取得

**【ターミナル2: Kafka / Flink CLI】**

JobManagerコンテナに入ります。

```bash
docker compose exec jobmanager bash
```

Savepointを取得します（ジョブIDは実際の値に置き換えてください）。

```bash
flink savepoint <JOB_ID> /tmp/savepoints
```

実行例：

```
Triggering savepoint for job b35139dca72a020a24349e376948b82b.
Waiting for response...
Savepoint completed. Path: file:/tmp/savepoints/savepoint-b35139-90fea20aed2f
You can resume your program from this savepoint with the run command.
```

表示されたSavepointのパスをメモしておきます。

### ステップ4: ジョブの停止

引き続きJobManagerコンテナ内で、ジョブを停止します。

```bash
flink cancel <JOB_ID>
```

実行例：

```
Cancelling job b35139dca72a020a24349e376948b82b.
Cancelled job b35139dca72a020a24349e376948b82b.
```

Flink Web UIでジョブのステータスが「CANCELED」になったことを確認します。

### ステップ5: 停止中にKafkaにデータを送信

ジョブが停止している間にKafkaにデータを送信します。

**【ターミナル2】**

JobManagerコンテナから抜けます。

```bash
exit
```

Kafkaにデータを送信します。

```bash
docker compose exec kafka /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic savepoint-events
```

以下のデータを送信します（10:01台のデータ）。

```json
{"event_id":"sp005","user_id":"u002","event_type":"view","event_time":"2025-12-30T10:01:10"}
{"event_id":"sp006","user_id":"u001","event_type":"click","event_time":"2025-12-30T10:01:30"}
{"event_id":"sp007","user_id":"u002","event_type":"purchase","event_time":"2025-12-30T10:01:50"}
```

ウィンドウを確定させるためのデータも送信します。

```json
{"event_id":"sp008","user_id":"u001","event_type":"click","event_time":"2025-12-30T10:03:00"}
```

`Ctrl+C`で終了します。

### ステップ6: Savepointからジョブを再開

**【ターミナル1: Flink ストリーミング】**

Savepointからジョブを再開するには、`execution.savepoint.path`を設定してからジョブを実行します。

```sql
-- Savepointパスを設定（実際のパスに置き換えてください）
SET 'execution.savepoint.path' = 'file:/tmp/savepoints/savepoint-b35139-90fea20aed2f';

-- チェックポイント間隔を設定
SET 'execution.checkpointing.interval' = '10s';

-- Kafkaソーステーブルを再作成（同じセッションなら既存テーブルを削除）
USE CATALOG default_catalog;

DROP TABLE IF EXISTS kafka_savepoint_events;

CREATE TABLE kafka_savepoint_events (
    event_id STRING,
    user_id STRING,
    event_type STRING,
    event_time TIMESTAMP(3),
    WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'savepoint-events',
    'properties.bootstrap.servers' = 'kafka:29092',
    'properties.group.id' = 'flink-savepoint-consumer',
    'scan.startup.mode' = 'earliest-offset',
    'format' = 'json',
    'json.timestamp-format.standard' = 'ISO-8601'
);

-- ジョブを再実行（Savepointから復元される）
INSERT INTO iceberg_catalog.demo.savepoint_test
SELECT 
    window_start,
    window_end,
    event_type,
    COUNT(*) as event_count
FROM TABLE(
    TUMBLE(TABLE kafka_savepoint_events, DESCRIPTOR(event_time), INTERVAL '1' MINUTE)
)
GROUP BY window_start, window_end, event_type;
```

### ステップ7: 結果の確認

**【ターミナル3: Flink バッチ】**

```sql
USE CATALOG iceberg_catalog;
USE demo;

SET 'execution.runtime-mode' = 'batch';
SELECT * FROM savepoint_test ORDER BY window_start, event_type;
```

結果例：

| window_start | window_end | event_type | event_count |
|--------------|------------|------------|-------------|
| 10:00:00 | 10:01:00 | click | 2 |
| 10:00:00 | 10:01:00 | view | 1 |
| 10:01:00 | 10:02:00 | click | 1 |
| 10:01:00 | 10:02:00 | purchase | 1 |
| 10:01:00 | 10:02:00 | view | 1 |

停止中に送信した10:01台のデータ（sp005, sp006, sp007）が正しく処理されています。

### 補足: 未確定のウィンドウについて

結果を見ると、ステップ2で送信したsp004（10:02:00）が表示されていません。これはウィンドウがまだ確定していないためです。

#### 送信したデータの整理

| イベント | event_time | ウィンドウ | 状態 |
|---------|------------|-----------|------|
| sp001〜sp003 | 10:00:10〜10:00:50 | 10:00〜10:01 | ✅ 確定 |
| sp004 | 10:02:00 | 10:02〜10:03 | ❌ 未確定 |
| sp005〜sp007 | 10:01:10〜10:01:50 | 10:01〜10:02 | ✅ 確定 |
| sp008 | 10:03:00 | 10:03〜10:04 | ❌ 未確定 |

#### ウォーターマークの計算

最後に送信したsp008（10:03:00）に基づくウォーターマークは以下の通りです。

```
ウォーターマーク = 10:03:00 - 5秒 = 10:02:55
```

ウォーターマークが10:02:55なので、10:02〜10:03のウィンドウ（終了時刻10:03:00）はまだ確定しません。

#### ウィンドウを確定させる

sp004が属する10:02〜10:03のウィンドウを確定させるには、ウォーターマークが10:03:00以上になるデータを送信します。

**【ターミナル2: Kafka】**

```bash
docker compose exec kafka /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic savepoint-events
```

```json
{"event_id":"sp009","user_id":"u001","event_type":"click","event_time":"2025-12-30T10:03:05"}
```

このデータのウォーターマークは `10:03:05 - 5秒 = 10:03:00` となり、10:02〜10:03のウィンドウが確定します。

**【ターミナル3: Flink バッチ】**

```sql
SELECT * FROM savepoint_test ORDER BY window_start, event_type;
```

結果例：

| window_start | window_end | event_type | event_count |
|--------------|------------|------------|-------------|
| 10:00:00 | 10:01:00 | click | 2 |
| 10:00:00 | 10:01:00 | view | 1 |
| 10:01:00 | 10:02:00 | click | 1 |
| 10:01:00 | 10:02:00 | purchase | 1 |
| 10:01:00 | 10:02:00 | view | 1 |
| 10:02:00 | 10:03:00 | click | 1 |

sp004が属する10:02〜10:03のウィンドウが確定し、結果に表示されました。

## Savepointに保存される情報

Savepointには以下の情報が保存されます。

| 情報 | 説明 |
|------|------|
| Kafkaオフセット | どこまで読んだか |
| ウィンドウの状態 | 集計中のデータ |
| シンク（Iceberg）の状態 | コミット状態 |

これにより、ジョブを停止・再開してもデータの欠損や重複なく処理を継続できます。

## まとめ

### Savepoint操作の流れ

```
1. Savepoint取得: flink savepoint <JOB_ID> <PATH>
2. ジョブ停止:    flink cancel <JOB_ID>
3. ジョブ再開:    SET 'execution.savepoint.path' = '<SAVEPOINT_PATH>';
                  INSERT INTO ... SELECT ...
```

### 重要なポイント

1. **Savepointの取得**: `flink savepoint`コマンドでジョブの状態を保存
2. **再開時の設定**: `execution.savepoint.path`でSavepointのパスを指定
3. **同じジョブ定義**: 再開時は同じSQLを実行する必要がある
4. **Kafkaオフセット**: Savepointに保存されるため、停止中のデータも正しく処理される

### 注意点

| 項目 | 内容 |
|------|------|
| ジョブの変更 | SQLを変更すると状態の互換性がなくなる場合がある |
| Savepointの保存先 | 本番環境ではS3などの永続ストレージを推奨 |
| 有効期限 | Savepointは明示的に削除するまで残る |

## 参考資料

- [Apache Flink公式ドキュメント - Savepoints](https://nightlies.apache.org/flink/flink-docs-release-1.18/docs/ops/state/savepoints/)
- [Flink CLI](https://nightlies.apache.org/flink/flink-docs-release-1.18/docs/deployment/cli/)
