---
title: "Debezium CDC入門 - Docker環境でMySQLからPostgreSQLへのリアルタイムデータ同期を実現する"
emoji: "🔄"
type: "tech"
topics: ["debezium", "cdc", "kafka", "mysql", "postgresql"]
published: false
---

## はじめに

分析基盤を構築する際、「アプリケーションに手を入れずにデータベースの変更を取得したい」という要件はよくあります。従来のポーリング方式では、更新頻度とデータ鮮度のトレードオフに悩まされることが多いですが、CDC（Change Data Capture）を使えば、データベースの変更をリアルタイムにキャプチャできます。

本記事では、OSSのCDCツールである **Debezium** を使って、MySQLの変更をKafka経由でPostgreSQLに同期する環境をDocker Composeで構築し、実際に動作検証を行います。

## CDCとは

CDC（Change Data Capture）は、データベースの変更（INSERT / UPDATE / DELETE）をキャプチャして、他のシステムに伝播させる技術です。

### なぜCDCが必要か

| 方式 | 仕組み | メリット | デメリット |
|------|--------|----------|-----------|
| ポーリング | 定期的にSELECTで差分取得 | 実装がシンプル | 遅延、DB負荷、削除検知が困難 |
| トリガー | DBトリガーで変更を記録 | リアルタイム | DB負荷、スキーマ変更が必要 |
| ログベースCDC | トランザクションログを読む | 低負荷、リアルタイム、削除も検知可能 | 実装が複雑 |

Debeziumは **ログベースCDC** を採用しており、MySQLのbinlogやPostgreSQLのWALを読み取ることで、アプリケーションに一切手を加えずにデータ変更をキャプチャできます。

## Debeziumとは

Debeziumは、Red Hatが開発するオープンソースのCDCプラットフォームです。

### 特徴

- **Kafka Connect** 上で動作するConnectorとして提供
- MySQL、PostgreSQL、MongoDB、Oracle、SQL Serverなど多数のDBに対応
- スナップショット機能（初回は既存データを全件取得）
- スキーマ変更の追跡
- Exactly-onceセマンティクスのサポート

### アーキテクチャ

```
┌─────────┐    ┌─────────────┐    ┌─────────┐    ┌──────────────┐    ┌────────────┐
│  MySQL  │    │  Debezium   │    │  Kafka  │    │  JDBC Sink   │    │ PostgreSQL │
│         │───▶│   Source    │───▶│ (Topic) │───▶│  Connector   │───▶│            │
│ binlog  │    │  Connector  │    │         │    │              │    │   tables   │
└─────────┘    └─────────────┘    └─────────┘    └──────────────┘    └────────────┘
               ├───────────────── Kafka Connect ──────────────────┤
```

## 検証環境構築

### 構成

Docker Composeで以下の5コンテナを構築します。

| コンテナ | 役割 | ポート |
|---------|------|--------|
| mysql | ソースDB | 3306 |
| kafka | メッセージブローカー（KRaftモード） | 9092, 29092 |
| connect | Kafka Connect + Debezium + JDBC Sink | 8083 |
| kafka-ui | Kafkaの可視化ツール | 8080 |
| postgres | シンクDB | 5432 |

### ファイル構成

```
debezium-cdc-demo/
├── docker-compose.yml
├── connect/
│   └── Dockerfile
├── mysql/
│   └── init/
│       └── 01_init.sql
└── postgres/
    └── init/
        └── 01_init.sql
```

### docker-compose.yml

```yaml
services:
  # ========================================
  # Source Database
  # ========================================
  mysql:
    image: mysql:8.0
    container_name: mysql
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: inventory
      MYSQL_USER: debezium
      MYSQL_PASSWORD: debezium
    command:
      - --server-id=1
      - --log-bin=mysql-bin
      - --binlog-format=ROW
      - --binlog-row-image=FULL
      - --gtid-mode=ON
      - --enforce-gtid-consistency=ON
    volumes:
      - ./mysql/init:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-prootpassword"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ========================================
  # Kafka with KRaft (No Zookeeper)
  # ========================================
  kafka:
    image: confluentinc/cp-kafka:7.6.0
    container_name: kafka
    ports:
      - "9092:9092"
      - "29092:29092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093,EXTERNAL://0.0.0.0:29092
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,EXTERNAL://localhost:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
    healthcheck:
      test: ["CMD-SHELL", "kafka-broker-api-versions --bootstrap-server localhost:9092 > /dev/null 2>&1"]
      interval: 10s
      timeout: 10s
      retries: 5
      start_period: 30s

  # ========================================
  # Kafka Connect with Debezium + JDBC Sink
  # ========================================
  connect:
    build: ./connect
    container_name: connect
    ports:
      - "8083:8083"
    environment:
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: connect_configs
      OFFSET_STORAGE_TOPIC: connect_offsets
      STATUS_STORAGE_TOPIC: connect_statuses
      BOOTSTRAP_SERVERS: kafka:9092
      KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      KEY_CONVERTER_SCHEMAS_ENABLE: "false"
      VALUE_CONVERTER_SCHEMAS_ENABLE: "false"
    depends_on:
      kafka:
        condition: service_healthy
      mysql:
        condition: service_healthy

  # ========================================
  # Kafka UI
  # ========================================
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
      KAFKA_CLUSTERS_0_KAFKACONNECT_0_NAME: debezium
      KAFKA_CLUSTERS_0_KAFKACONNECT_0_ADDRESS: http://connect:8083
    depends_on:
      - kafka
      - connect

  # ========================================
  # Sink Database
  # ========================================
  postgres:
    image: postgres:15
    container_name: postgres
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: cdc_sink
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - ./postgres/init:/docker-entrypoint-initdb.d
```

### connect/Dockerfile

Debeziumイメージに JDBC Sink Connector を追加します。

```dockerfile
FROM debezium/connect:2.5

# JDBC Sink Connector をダウンロードしてインストール
RUN mkdir -p /kafka/connect/kafka-connect-jdbc && \
    cd /kafka/connect/kafka-connect-jdbc && \
    curl -sO https://packages.confluent.io/maven/io/confluent/kafka-connect-jdbc/10.7.4/kafka-connect-jdbc-10.7.4.jar && \
    curl -sO https://jdbc.postgresql.org/download/postgresql-42.7.1.jar
```

### mysql/init/01_init.sql

```sql
-- Debeziumユーザーに必要な権限を付与
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'debezium'@'%';
FLUSH PRIVILEGES;

USE inventory;

-- 顧客テーブル（DATETIME型を使用）
CREATE TABLE customers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- サンプルデータ
INSERT INTO customers (first_name, last_name, email) VALUES
    ('Taro', 'Yamada', 'taro.yamada@example.com'),
    ('Hanako', 'Suzuki', 'hanako.suzuki@example.com'),
    ('Ichiro', 'Tanaka', 'ichiro.tanaka@example.com');
```

### postgres/init/01_init.sql

```sql
CREATE SCHEMA IF NOT EXISTS cdc;

CREATE TABLE cdc.customers (
    id INT PRIMARY KEY,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    email VARCHAR(255),
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    __op VARCHAR(10),
    __table VARCHAR(100),
    __source_ts_ms BIGINT,
    __deleted VARCHAR(10)
);
```

### 起動

```bash
docker compose up -d --build
docker compose ps  # 全コンテナがhealthyになるまで待つ
```

全コンテナが起動すると以下のようになります：

```
NAME       IMAGE                             COMMAND                  SERVICE    CREATED          STATUS                    PORTS
connect    debezium-cdc-demo-final-connect   "/docker-entrypoint.…"   connect    22 seconds ago   Up 7 seconds              0.0.0.0:8083->8083/tcp
kafka      confluentinc/cp-kafka:7.6.0       "/etc/confluent/dock…"   kafka      23 seconds ago   Up 21 seconds (healthy)   0.0.0.0:9092->9092/tcp, 0.0.0.0:29092->29092/tcp
kafka-ui   provectuslabs/kafka-ui:latest     "/bin/sh -c 'java --…"   kafka-ui   22 seconds ago   Up 7 seconds              0.0.0.0:8080->8080/tcp
mysql      mysql:8.0                         "docker-entrypoint.s…"   mysql      23 seconds ago   Up 21 seconds (healthy)   0.0.0.0:3306->3306/tcp
postgres   postgres:15                       "docker-entrypoint.s…"   postgres   23 seconds ago   Up 21 seconds             0.0.0.0:5432->5432/tcp
```

## CDC動作検証

### Source Connector の登録

```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "inventory-connector",
    "config": {
      "connector.class": "io.debezium.connector.mysql.MySqlConnector",
      "tasks.max": "1",
      "database.hostname": "mysql",
      "database.port": "3306",
      "database.user": "debezium",
      "database.password": "debezium",
      "database.server.id": "184054",
      "topic.prefix": "dbserver1",
      "database.include.list": "inventory",
      "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
      "schema.history.internal.kafka.topic": "schema-changes.inventory",
      "include.schema.changes": "true",
      "time.precision.mode": "connect"
    }
  }'
```

`time.precision.mode: connect` については後述します。

### Sink Connector の登録

```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "jdbc-sink-customers",
    "config": {
      "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
      "tasks.max": "1",
      "connection.url": "jdbc:postgresql://postgres:5432/cdc_sink",
      "connection.user": "postgres",
      "connection.password": "postgres",
      "topics": "dbserver1.inventory.customers",
      "table.name.format": "cdc.customers",
      "insert.mode": "upsert",
      "pk.mode": "record_key",
      "pk.fields": "id",
      "delete.enabled": "true",
      "auto.create": "false",
      "consumer.auto.offset.reset": "earliest",
      "transforms": "unwrap,extractKey",
      "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
      "transforms.unwrap.drop.tombstones": "false",
      "transforms.unwrap.delete.handling.mode": "rewrite",
      "transforms.unwrap.add.fields": "op,table,source.ts_ms",
      "transforms.extractKey.type": "org.apache.kafka.connect.transforms.ExtractField$Key",
      "transforms.extractKey.field": "id"
    }
  }'
```

#### 設定のポイント

**削除の同期に必要な設定**

`delete.enabled: true` でDELETE操作を同期する場合、以下の設定が必須です：

- `transforms.unwrap.drop.tombstones: false` - Debeziumが送るTombstoneレコード（Valueがnullのメッセージ）を破棄せずに処理します。これを `true` にすると削除イベントが正しく伝播されません。
- `transforms.unwrap.delete.handling.mode: rewrite` - 削除レコードを `__deleted: true` フラグ付きで書き換えます。

**主キーの設定**

`pk.mode: record_key` は、KafkaメッセージのKey部分を使用してPostgreSQLの主キーを特定する設定です。Debeziumはデフォルトでテーブルの主キーをメッセージのKeyに含めるため、この設定で正しく動作します。`extractKey` トランスフォームでKeyから `id` フィールドを抽出しています。

### 動作確認

Kafka UI（http://localhost:8080）で `dbserver1.inventory.customers` トピックにメッセージが流れていることを確認できます。

PostgreSQLでデータを確認：

```bash
docker compose exec postgres psql -U postgres -d cdc_sink -c "
SELECT id, first_name, last_name, created_at, __op FROM cdc.customers ORDER BY id;
"
```

初期データ（スナップショット）が同期されていることを確認できます：

```
 id | first_name | last_name |     created_at      | __op
----+------------+-----------+---------------------+------
  1 | Taro       | Yamada    | 2026-01-17 01:05:39 | r
  2 | Hanako     | Suzuki    | 2026-01-17 01:05:39 | r
  3 | Ichiro     | Tanaka    | 2026-01-17 01:05:39 | r
(3 rows)
```

### INSERT / UPDATE / DELETE の検証

```bash
# INSERT
docker compose exec mysql mysql -u debezium -pdebezium inventory -e "
INSERT INTO customers (first_name, last_name, email) 
VALUES ('Jiro', 'Sato', 'jiro.sato@example.com');
"

# UPDATE
docker compose exec mysql mysql -u debezium -pdebezium inventory -e "
UPDATE customers SET first_name = 'Taro-Updated' WHERE id = 1;
"

# DELETE
docker compose exec mysql mysql -u debezium -pdebezium inventory -e "
DELETE FROM customers WHERE id = 3;
"
```

各操作後にPostgreSQLを確認すると、リアルタイムで同期されていることがわかります：

```bash
docker compose exec postgres psql -U postgres -d cdc_sink -c "
SELECT id, first_name, last_name, __op, __deleted FROM cdc.customers ORDER BY id;
"
```

```
 id |  first_name  | last_name | __op | __deleted
----+--------------+-----------+------+-----------
  1 | Taro-Updated | Yamada    | u    | false
  2 | Hanako       | Suzuki    | r    | false
  4 | Jiro         | Sato      | c    | false
(3 rows)
```

id=3（Ichiro）は DELETE により削除され、id=1 は UPDATE で `__op` が `u` に、id=4 は INSERT で `__op` が `c` になっています。

`__op` カラムで操作種別を確認できます：

| 値 | 意味 |
|----|------|
| r | Read（スナップショット） |
| c | Create（INSERT） |
| u | Update（UPDATE） |
| d | Delete（DELETE） |

## 日付型の変換について

MySQLの日付型によって、Debeziumの出力形式と必要な設定が異なります。

### 検証結果

| MySQL 型 | time.precision.mode | 設定の効果 | Kafka 出力 | SMT 必要？ |
|----------|---------------------|-----------|------------|-----------|
| DATETIME | なし | - | INT64（エポックミリ秒） | ✓ 必要 |
| DATETIME | connect | ✓ 効く | Timestamp（論理型） | ✗ 不要 |
| TIMESTAMP | なし | - | STRING（ISO 8601） | ✓ 必要 |
| TIMESTAMP | connect | ✗ 効かない | STRING（ISO 8601） | ✓ 必要 |

### 解説

**DATETIME型の場合**

`time.precision.mode: connect` を指定すると、Kafka Connectの論理型（Timestamp）として出力されるため、JDBC Sink Connectorが自動的にPostgreSQLのTIMESTAMP型に変換してくれます。

**TIMESTAMP型の場合**

`time.precision.mode: connect` を指定しても、DebeziumはMySQLのTIMESTAMP型を常に `io.debezium.time.ZonedTimestamp`（ISO 8601文字列）として出力します。そのため、Sink側でSMT（TimestampConverter）による変換が必要です。

### TIMESTAMP型の場合のSink Connector設定

```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "jdbc-sink-customers",
    "config": {
      "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
      ... 省略 ...
      "transforms": "unwrap,extractKey,convertCreatedAt,convertUpdatedAt",
      ... 省略 ...
      "transforms.convertCreatedAt.type": "org.apache.kafka.connect.transforms.TimestampConverter$Value",
      "transforms.convertCreatedAt.field": "created_at",
      "transforms.convertCreatedAt.format": "yyyy-MM-dd'\''T'\''HH:mm:ss'\''Z'\''",
      "transforms.convertCreatedAt.target.type": "Timestamp",
      "transforms.convertUpdatedAt.type": "org.apache.kafka.connect.transforms.TimestampConverter$Value",
      "transforms.convertUpdatedAt.field": "updated_at",
      "transforms.convertUpdatedAt.format": "yyyy-MM-dd'\''T'\''HH:mm:ss'\''Z'\''",
      "transforms.convertUpdatedAt.target.type": "Timestamp"
    }
  }'
```

### 推奨設定

| MySQL の型 | 推奨設定 |
|------------|---------|
| DATETIME のみ | Source に `time.precision.mode: connect`（シンプル） |
| TIMESTAMP のみ | Sink に SMT（TimestampConverter）を追加 |
| 混在 | Sink に SMT で統一するのが安全 |

## ユースケース

Debezium CDCは以下のようなユースケースで活用できます。

### 分析基盤へのデータ連携

アプリケーションDBの変更を分析基盤（DWH、データレイク）にリアルタイムで連携。ETLバッチ処理の待ち時間を削減できます。

### マイクロサービス間のデータ同期

サービス間でデータを同期する際、直接API呼び出しではなくCDC経由にすることで、疎結合なアーキテクチャを実現できます。

### キャッシュの更新

DBの変更をキャッシュ（Redis等）に即座に反映させることで、キャッシュの整合性を保てます。

### 監査ログ

すべてのデータ変更をKafkaに記録することで、変更履歴の完全な監査ログを構築できます。

## まとめ

本記事では、Debezium CDCを使ってMySQLからPostgreSQLへのリアルタイムデータ同期を実現しました。

### 学んだこと

- CDCの概念とログベースCDCのメリット
- Docker ComposeでのDebezium環境構築（KRaftモードのKafka）
- Source / Sink Connectorの設定方法
- 日付型の変換における注意点と対処法

### 次回予告

次回は Elasticsearch + Kibana を追加して、CDCデータのリアルタイム可視化を試してみようと思います。

## 参考リンク

- [Debezium Documentation](https://debezium.io/documentation/)
- [Debezium Connector for MySQL](https://debezium.io/documentation/reference/stable/connectors/mysql.html)
- [Confluent JDBC Sink Connector](https://docs.confluent.io/kafka-connectors/jdbc/current/sink-connector/overview.html)
- [Kafka Connect Transforms](https://docs.confluent.io/platform/current/connect/transforms/overview.html)
