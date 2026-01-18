---
title: "Debezium CDC 実践編 - Elasticsearch + Kibanaでリアルタイム可視化"
emoji: "📊"
type: "tech"
topics: ["debezium", "cdc", "elasticsearch", "kibana", "kafka"]
published: false
---

## はじめに

前回の記事では、Debezium を使って MySQL の変更データを PostgreSQL にリアルタイム同期できるか検証しました。

https://zenn.dev/toshiro3/articles/debezium-cdc-introduction

今回は、同じ CDC パイプラインを拡張して、**Elasticsearch + Kibana** でリアルタイム可視化ができるか検証してみます。

### 今回のゴール

- MySQL の変更を Elasticsearch に自動同期
- Kibana ダッシュボードでリアルタイムにデータを可視化
- INSERT / UPDATE が即座に反映されることを確認

## アーキテクチャ

```
┌─────────┐    ┌───────────┐    ┌─────────┐    ┌────────────────┐    ┌─────────────────┐
│  MySQL  │───▶│  Debezium │───▶│  Kafka  │───▶│ Elasticsearch  │───▶│     Kibana      │
│ (binlog)│    │  Source   │    │ (Topic) │    │ Sink Connector │    │  (ダッシュボード) │
└─────────┘    └───────────┘    └─────────┘    └────────────────┘    └─────────────────┘
                                      │
                                      ▼
                               ┌────────────┐
                               │ PostgreSQL │  ※ 前回の記事で構築済み
                               │    Sink    │
                               └────────────┘
```

前回の構成に **Elasticsearch** と **Kibana** を追加し、Kafka から Elasticsearch への Sink Connector を設定します。

## 環境構築

### ディレクトリ構成

```
debezium-cdc-elasticsearch-kibana/
├── docker-compose.yml
├── connect/
│   ├── Dockerfile
│   └── pom.xml
├── mysql/
│   └── init/
│       └── 01_init.sql
└── postgres/
    └── init/
        └── 01_init.sql
```

### 前回からの主な変更点

1. **Elasticsearch / Kibana コンテナの追加**
2. **Kafka Connect の Dockerfile 変更**（Maven を使ったマルチステージビルド）
3. **Elasticsearch Sink Connector 用の依存関係追加**

### Dockerfile（connect/Dockerfile）

前回は curl で直接 JAR をダウンロードしていましたが、Elasticsearch Sink Connector は依存関係が複雑なため、**Maven を使ったマルチステージビルド**に変更しました。

```dockerfile
# Stage 1: Maven で依存関係をダウンロード
FROM maven:3.9-eclipse-temurin-21 AS builder

WORKDIR /build

COPY pom.xml .

# 依存関係をダウンロード
RUN mvn dependency:copy-dependencies -DoutputDirectory=/build/libs

# Stage 2: Debezium Connect に jar を追加
FROM debezium/connect:2.7.3.Final

USER root

# Elasticsearch Sink Connector（全ての依存関係をまとめてコピー）
RUN mkdir -p /kafka/connect/kafka-connect-elasticsearch
COPY --from=builder /build/libs/*.jar /kafka/connect/kafka-connect-elasticsearch/

# JDBC Sink Connector（必要な jar のみコピー）
RUN mkdir -p /kafka/connect/kafka-connect-jdbc
COPY --from=builder /build/libs/kafka-connect-jdbc-*.jar /kafka/connect/kafka-connect-jdbc/
COPY --from=builder /build/libs/postgresql-*.jar /kafka/connect/kafka-connect-jdbc/

USER kafka
```

:::message
Maven の `dependency:copy-dependencies` で全ての依存関係が `/build/libs/` に集約されるため、`kafka-connect-elasticsearch` ディレクトリには JDBC 関連の JAR も含まれています。本来は不要ですが、動作検証の観点では問題ないと判断しています。
:::

### pom.xml（connect/pom.xml）

Maven で必要な Connector と依存関係を定義します。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>kafka-connect-deps</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <dependencies>
        <!-- Elasticsearch Sink Connector -->
        <dependency>
            <groupId>io.confluent</groupId>
            <artifactId>kafka-connect-elasticsearch</artifactId>
            <version>15.0.0</version>
        </dependency>
        <!-- JDBC Sink Connector -->
        <dependency>
            <groupId>io.confluent</groupId>
            <artifactId>kafka-connect-jdbc</artifactId>
            <version>10.7.4</version>
        </dependency>
        <!-- PostgreSQL JDBC Driver -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <version>42.7.1</version>
        </dependency>
    </dependencies>

    <repositories>
        <repository>
            <id>confluent</id>
            <url>https://packages.confluent.io/maven/</url>
        </repository>
    </repositories>
</project>
```

:::message
Confluent の Kafka Connect プラグインは、Maven Central ではなく Confluent のリポジトリで公開されているため、`<repositories>` の設定が必要です。
:::

### docker-compose.yml

前回の構成に Elasticsearch と Kibana を追加します。

:::message
Kafka 3系（KRaft モード）を使用しているため、Zookeeper は不要です。
:::

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
      # KRaft設定
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
      # リスナー設定
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093,EXTERNAL://0.0.0.0:29092
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,EXTERNAL://localhost:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      # その他
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
  # Kafka Connect with Debezium + Sink Connectors
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
      elasticsearch:
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
  # Sink Database (PostgreSQL)
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

  # ========================================
  # Elasticsearch
  # ========================================
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    container_name: elasticsearch
    ports:
      - "9200:9200"
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - xpack.security.enrollment.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:9200/_cluster/health | grep -q 'green\\|yellow'"]
      interval: 10s
      timeout: 10s
      retries: 10
      start_period: 30s

  # ========================================
  # Kibana
  # ========================================
  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      elasticsearch:
        condition: service_healthy
```

#### docker-compose.yml のポイント

| 項目 | 説明 |
|------|------|
| **KRaft モード** | Kafka 3系では Zookeeper が不要。`KAFKA_PROCESS_ROLES: broker,controller` で設定 |
| **healthcheck** | 各サービスの起動完了を確認してから依存サービスを起動 |
| **condition: service_healthy** | healthcheck が成功するまで待機してから起動 |

### 起動

```bash
docker compose up -d --build
```

全てのコンテナが起動するまで待ちます（特に Elasticsearch は起動に時間がかかります）。

```bash
docker compose ps
```

## Connector 設定

### 1. Source Connector（MySQL → Kafka）

前回と同じ設定です。

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

### 2. JDBC Sink Connector（Kafka → PostgreSQL）

前回と同じ設定です。

```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "jdbc-sink-connector",
    "config": {
      "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
      "tasks.max": "1",
      "connection.url": "jdbc:postgresql://postgres:5432/cdc_sink",
      "connection.user": "postgres",
      "connection.password": "postgres",
      "topics": "dbserver1.inventory.customers",
      "table.name.format": "customers_cdc",
      "insert.mode": "upsert",
      "pk.mode": "record_key",
      "pk.fields": "id",
      "auto.create": "true",
      "auto.evolve": "true",
      "transforms": "unwrap",
      "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
      "transforms.unwrap.drop.tombstones": "true"
    }
  }'
```

### 3. Elasticsearch Sink Connector（Kafka → Elasticsearch）

今回追加する Connector です。

```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "elasticsearch-sink-customers",
    "config": {
      "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
      "tasks.max": "1",
      "connection.url": "http://elasticsearch:9200",
      "topics": "dbserver1.inventory.customers",
      "key.ignore": "false",
      "schema.ignore": "true",
      "behavior.on.null.values": "delete",
      "transforms": "unwrap,extractKey",
      "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
      "transforms.unwrap.drop.tombstones": "false",
      "transforms.unwrap.delete.handling.mode": "none",
      "transforms.extractKey.type": "org.apache.kafka.connect.transforms.ExtractField$Key",
      "transforms.extractKey.field": "id"
    }
  }'
```

#### 設定のポイント

| 設定 | 値 | 説明 |
|------|-----|------|
| `key.ignore` | `false` | Kafka のキーを Elasticsearch の `_id` として使用 |
| `schema.ignore` | `true` | スキーマを無視してJSONとして処理 |
| `behavior.on.null.values` | `delete` | DELETE イベント時にドキュメントを削除 |
| `transforms.unwrap` | ExtractNewRecordState | Debezium のエンベロープから `after` フィールドを抽出 |
| `transforms.unwrap.drop.tombstones` | `false` | tombstone レコードを保持（DELETE検知用） |
| `transforms.unwrap.delete.handling.mode` | `none` | DELETE時はnullを渡す（behavior.on.null.valuesで処理） |
| `transforms.extractKey` | ExtractField$Key | キーから `id` フィールドを抽出 |

### Connector 状態確認

```bash
curl -s http://localhost:8083/connectors?expand=status | jq
```

3つの Connector が `RUNNING` になっていれば OK です。

## Elasticsearch での動作確認

Kibana でダッシュボードを作成する前に、CLI で CDC が正しく動作しているか確認します。

### INSERT の確認

MySQL にデータを追加し、Elasticsearch に同期されるか確認します。

```bash
# INSERT
docker compose exec mysql mysql -u debezium -pdebezium inventory -e "
INSERT INTO customers (first_name, last_name, email) 
VALUES ('Jiro', 'Sato', 'jiro.sato@example.com');
"

# 確認
sleep 2
curl -s 'http://localhost:9200/dbserver1.inventory.customers/_search?pretty' | jq '.hits.hits[]._source.first_name'
```

追加したデータが Elasticsearch に反映されていれば成功です。

### UPDATE の確認

```bash
# UPDATE
docker compose exec mysql mysql -u debezium -pdebezium inventory -e "
UPDATE customers SET first_name = 'Taro-Updated' WHERE id = 1;
"

# 確認
sleep 2
curl -s 'http://localhost:9200/dbserver1.inventory.customers/_doc/1?pretty'
```

`first_name` が `Taro-Updated` に変わっていることを確認します。

### DELETE の確認

```bash
# DELETE
docker compose exec mysql mysql -u debezium -pdebezium inventory -e "
DELETE FROM customers WHERE id = 3;
"

# 確認
sleep 2
curl -s 'http://localhost:9200/dbserver1.inventory.customers/_search?pretty' | jq '.hits.total.value'
```

ドキュメント数が減っていれば、DELETE も正しく同期されています。

## Kibana でダッシュボード作成

### 1. Kibana にアクセス

```
http://localhost:5601
```

### 2. Data View の作成

1. 左メニュー **≡** → **Management** → **Stack Management**
2. **Kibana** → **Data Views**
3. **Create data view** をクリック
4. 設定：
   - **Name**: `customers`
   - **Index pattern**: `dbserver1.inventory.customers`
   - **Timestamp field**: 選択しない（または `updated_at`）
5. **Save data view to Kibana**

### 3. Discover でデータ確認

1. 左メニュー **≡** → **Analytics** → **Discover**
2. 作成した Data View `customers` を選択
3. MySQL から同期されたデータが表示されることを確認

![Discover画面](/images/debezium-cdc-elasticsearch-kibana/discover.png)

### 4. ダッシュボード作成

1. 左メニュー **≡** → **Analytics** → **Dashboard**
2. **Create dashboard** をクリック

#### 可視化1: 顧客一覧テーブル

1. **Create visualization** をクリック
2. **Visualization type**: `Table`
3. 左側のフィールドから **Rows** に追加：
   - `first_name`
   - `last_name`
   - `email`
4. **Save and return**

#### 可視化2: レコード数メトリック

1. 再度 **Create visualization** をクリック
2. **Visualization type**: `Metric`
3. **Primary metric**: `Count`（デフォルト）
4. **Save and return**

#### ダッシュボードを保存

1. 右上の **Save** をクリック
2. タイトル: `CDC Customers Dashboard`
3. **Save**

## リアルタイム同期のデモ

ダッシュボードを作成した状態で、MySQL のデータを変更し、Kibana にリアルタイムで反映されることを確認します。

### INSERT の確認

```bash
docker compose exec mysql mysql -u debezium -pdebezium inventory -e "
INSERT INTO customers (first_name, last_name, email) 
VALUES ('Saburo', 'Tanaka', 'saburo.tanaka@example.com');
"
```

Kibana ダッシュボードの **🔄（リフレッシュ）** をクリックすると、追加されたデータが表示されます。

![INSERT後のダッシュボード](/images/debezium-cdc-elasticsearch-kibana/after-insert.png)

### UPDATE の確認

```bash
docker compose exec mysql mysql -u debezium -pdebezium inventory -e "
UPDATE customers SET first_name = 'Hanako-Updated' WHERE id = 2;
"
```

再度リフレッシュすると、`Hanako` が `Hanako-Updated` に変わっていることが確認できます。

![UPDATE後のダッシュボード](/images/debezium-cdc-elasticsearch-kibana/after-update.png)

## PostgreSQL vs Elasticsearch の使い分け

PostgreSQL Sink と Elasticsearch Sink の使い分けを整理します。

| 観点 | PostgreSQL | Elasticsearch |
|------|------------|---------------|
| **用途** | トランザクション処理、レポート | 全文検索、リアルタイム分析 |
| **クエリ** | SQL | DSL / KQL |
| **可視化** | 外部ツールが必要 | Kibana が統合済み |
| **スケーラビリティ** | 垂直スケール中心 | 水平スケールが容易 |
| **データの一貫性** | ACID 準拠 | 結果整合性（デフォルト1秒の refresh interval） |
| **適したケース** | マスタデータ同期、バックアップ | ログ分析、ダッシュボード |

:::message
Elasticsearch は結果整合性のため、データ反映に若干のラグがあります。検証コマンドで `sleep 2` を入れているのはこのためです。
:::

## まとめ

Debezium CDC のパイプラインに Elasticsearch + Kibana を追加し、リアルタイム可視化ができることを確認しました。

### 今回のポイント

- **Maven マルチステージビルド**で Kafka Connect プラグインの依存関係を解決
- **Elasticsearch Sink Connector** で CDC イベントを Elasticsearch に同期
- **Kibana ダッシュボード**で INSERT / UPDATE がリアルタイムに反映されることを確認

CDC パイプラインを活用することで、データベースの変更をリアルタイムに様々なシステムへ連携できます。分析基盤やマイクロサービス間のデータ同期など、幅広いユースケースに応用できるでしょう。
