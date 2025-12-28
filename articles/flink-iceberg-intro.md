---
title: "Flink SQL入門 - DockerでIcebergテーブルを操作する"
emoji: "🌊"
type: "tech"
topics: ["ApacheFlink", "ApacheIceberg", "Docker", "DataEngineering", "SQL"]
published: false
---

## はじめに

Apache Flinkは、分散処理エンジンです。リアルタイムデータ処理に強みを持ち、バッチ処理も統一的に扱えます。

本記事では、Docker ComposeでFlink + Iceberg環境を構築し、Flink SQLでIcebergテーブルを操作する基本を体験します。

### この記事で扱う内容

- Flinkの概要とSparkとの違い
- Docker Composeでの環境構築
- Flink SQLでの基本操作（テーブル作成、INSERT、SELECT）
- メタデータテーブルとタイムトラベルクエリ
- UPSERTモード

## Apache Flinkとは

リアルタイムに流れてくる膨大なデータを、高い信頼性とスピードを保ちながら、過去の状態も考慮して処理・分析できる「分散処理エンジン」です。

公式サイトでは「Stateful Computations over Data Streams」と表現されています。

| 特徴 | 説明 |
|------|------|
| **Correctness** | Exactly-once保証、イベント時間処理、遅延データのハンドリング |
| **Layered APIs** | SQL、DataStream API、ProcessFunctionの階層化されたAPI |
| **Scalability** | スケールアウトアーキテクチャ、大規模な状態管理 |
| **Performance** | 低レイテンシ、高スループット、インメモリ処理 |

### SparkとFlinkの違い

| 観点 | Spark | Flink |
|------|-------|-------|
| **設計思想** | バッチ処理が基本、ストリームは後付け | ストリーム処理が基本、バッチは特殊ケース |
| **処理モデル** | マイクロバッチ | 真のストリーム処理 |
| **レイテンシ** | 秒単位 | ミリ秒単位 |
| **得意分野** | バッチETL、機械学習 | リアルタイム処理、イベント駆動 |

### Hadoopとの関係

FlinkはHadoopエコシステムから**独立したプロジェクト**です。

- Hadoopに**依存せず**単独で動作可能
- ただし、HDFSやS3へのアクセスにHadoopのライブラリを**利用**することがある
- YARNやHDFSと**連携**できるが、必須ではない

今回の環境でもS3（MinIO）へのアクセスにHadoopのクライアントライブラリを使用しています。

## 環境構築

本記事で使用する環境はGitHubリポジトリで公開しています。

https://github.com/toshiro3/flink-iceberg-lab

```bash
git clone https://github.com/toshiro3/flink-iceberg-lab.git
cd flink-iceberg-lab
```

### アーキテクチャ

```
┌─────────────────────────────────────────────────────────────┐
│                    Flink Cluster                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │ JobManager  │    │ TaskManager │    │ SQL Client  │     │
│  │  (Master)   │◄───│  (Worker)   │    │  (CLI)      │     │
│  │  :8081      │    │             │    │             │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
                   ┌───────────────┐
                   │  REST Catalog │
                   │    :8181      │
                   └───────┬───────┘
                           │
                           ▼
                   ┌───────────────┐
                   │     MinIO     │
                   │  S3 API: 9000 │
                   │  Console:9001 │
                   └───────────────┘
```

### Flinkクラスタの構成

| コンポーネント | 役割 |
|---------------|------|
| **JobManager** | クラスタのマスターノード。ジョブのスケジューリング、チェックポイント管理を担当 |
| **TaskManager** | クラスタのワーカーノード。実際のデータ処理タスクを実行 |
| **SQL Client** | 対話的にFlink SQLを実行するCLI |

### ディレクトリ構成

```
flink-iceberg-lab/
├── docker-compose.yml
├── flink/
│   └── Dockerfile
├── sql/
├── minio-data/
└── warehouse/
```

### docker-compose.yml

```yaml
services:
  # Flink JobManager: クラスタのマスターノード
  jobmanager:
    build:
      context: ./flink
      dockerfile: Dockerfile
    container_name: jobmanager
    ports:
      - "8081:8081"
    command: jobmanager
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        state.backend: rocksdb
        state.checkpoints.dir: file:///tmp/flink-checkpoints
        state.savepoints.dir: file:///tmp/flink-savepoints
      - AWS_ACCESS_KEY_ID=admin
      - AWS_SECRET_ACCESS_KEY=password
      - AWS_REGION=us-east-1
    depends_on:
      minio-init:
        condition: service_completed_successfully
    networks:
      - flink-net

  # Flink TaskManager: クラスタのワーカーノード
  taskmanager:
    build:
      context: ./flink
      dockerfile: Dockerfile
    container_name: taskmanager
    depends_on:
      - jobmanager
    command: taskmanager
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        taskmanager.numberOfTaskSlots: 4
        state.backend: rocksdb
        state.checkpoints.dir: file:///tmp/flink-checkpoints
        state.savepoints.dir: file:///tmp/flink-savepoints
      - AWS_ACCESS_KEY_ID=admin
      - AWS_SECRET_ACCESS_KEY=password
      - AWS_REGION=us-east-1
    networks:
      - flink-net

  # Flink SQL Client
  sql-client:
    build:
      context: ./flink
      dockerfile: Dockerfile
    container_name: sql-client
    depends_on:
      - jobmanager
      - rest-catalog
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        rest.address: jobmanager
        rest.port: 8081
      - AWS_ACCESS_KEY_ID=admin
      - AWS_SECRET_ACCESS_KEY=password
      - AWS_REGION=us-east-1
    networks:
      - flink-net
    stdin_open: true
    tty: true

  # REST Catalog: Icebergカタログサーバー
  rest-catalog:
    image: tabulario/iceberg-rest:1.6.0
    container_name: rest-catalog
    ports:
      - "8181:8181"
    environment:
      - AWS_ACCESS_KEY_ID=admin
      - AWS_SECRET_ACCESS_KEY=password
      - AWS_REGION=us-east-1
      - CATALOG_WAREHOUSE=s3://warehouse/
      - CATALOG_IO__IMPL=org.apache.iceberg.aws.s3.S3FileIO
      - CATALOG_S3_ENDPOINT=http://minio:9000
      - CATALOG_S3_PATH__STYLE__ACCESS=true
    depends_on:
      minio-init:
        condition: service_completed_successfully
    networks:
      - flink-net
    restart: on-failure

  # MinIO: S3互換オブジェクトストレージ
  minio:
    image: minio/minio:latest
    container_name: minio
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      - MINIO_ROOT_USER=admin
      - MINIO_ROOT_PASSWORD=password
    command: server /data --console-address ":9001"
    volumes:
      - ./minio-data:/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - flink-net

  # MinIO初期設定
  minio-init:
    image: minio/mc:latest
    container_name: minio-init
    depends_on:
      minio:
        condition: service_healthy
    entrypoint: >
      /bin/sh -c "
      mc alias set myminio http://minio:9000 admin password;
      mc mb myminio/warehouse --ignore-existing;
      mc anonymous set public myminio/warehouse;
      exit 0;
      "
    networks:
      - flink-net

networks:
  flink-net:
    driver: bridge
```

### flink/Dockerfile

Flink公式イメージにIcebergとHadoopのJARを追加しています。

```dockerfile
FROM flink:1.18

ENV HADOOP_VERSION=3.3.4

# Iceberg Flink Runtime
RUN curl -L -o /opt/flink/lib/iceberg-flink-runtime-1.18-1.6.1.jar \
    https://repo1.maven.org/maven2/org/apache/iceberg/iceberg-flink-runtime-1.18/1.6.1/iceberg-flink-runtime-1.18-1.6.1.jar

# AWS Bundle（S3アクセス用）
RUN curl -L -o /opt/flink/lib/iceberg-aws-bundle-1.6.1.jar \
    https://repo1.maven.org/maven2/org/apache/iceberg/iceberg-aws-bundle/1.6.1/iceberg-aws-bundle-1.6.1.jar

# Flink S3 Filesystem
RUN mkdir -p /opt/flink/plugins/s3-fs-hadoop && \
    cp /opt/flink/opt/flink-s3-fs-hadoop-*.jar /opt/flink/plugins/s3-fs-hadoop/

# Hadoop Client（Icebergの依存関係）
RUN curl -L -o /opt/flink/lib/hadoop-client-api-${HADOOP_VERSION}.jar \
    https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-client-api/${HADOOP_VERSION}/hadoop-client-api-${HADOOP_VERSION}.jar

RUN curl -L -o /opt/flink/lib/hadoop-client-runtime-${HADOOP_VERSION}.jar \
    https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-client-runtime/${HADOOP_VERSION}/hadoop-client-runtime-${HADOOP_VERSION}.jar

# Hadoop AWS（S3A用）
RUN curl -L -o /opt/flink/lib/hadoop-aws-${HADOOP_VERSION}.jar \
    https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-aws/${HADOOP_VERSION}/hadoop-aws-${HADOOP_VERSION}.jar

# AWS SDK Bundle
RUN curl -L -o /opt/flink/lib/aws-java-sdk-bundle-1.12.648.jar \
    https://repo1.maven.org/maven2/com/amazonaws/aws-java-sdk-bundle/1.12.648/aws-java-sdk-bundle-1.12.648.jar
```

:::message
IcebergはHadoopのファイルシステム抽象化を内部で使用しているため、Hadoopに依存しないFlinkでもHadoopのクライアントライブラリが必要です。
:::

### 起動

```bash
docker compose up -d
```

初回起動時はDockerイメージのビルド（JARのダウンロード）に数分かかります。

起動後、以下のURLでアクセスできます。

| サービス | URL | 用途 |
|---------|-----|------|
| Flink Web UI | http://localhost:8081 | ジョブ監視 |
| MinIO Console | http://localhost:9001 | ストレージ管理（admin/password） |

:::message alert
TaskManagerが起動していない場合があります。Flink Web UIの「Task Managers」タブで確認し、表示されていなければ以下を実行してください。

```bash
docker compose up -d taskmanager
```
:::

## SQL Clientの起動

```bash
docker compose run --rm sql-client /opt/flink/bin/sql-client.sh
```

以下のようなプロンプトが表示されます。

```
Flink SQL>
```

:::message
Flink SQLのコマンドはすべてセミコロン（`;`）が必要です。終了する場合は `QUIT;` を実行します。
:::

## Icebergカタログへの接続

REST Catalogを登録します。

```sql
CREATE CATALOG iceberg_catalog WITH (
    'type' = 'iceberg',
    'catalog-type' = 'rest',
    'uri' = 'http://rest-catalog:8181',
    'warehouse' = 's3://warehouse',
    's3.endpoint' = 'http://minio:9000',
    's3.access-key-id' = 'admin',
    's3.secret-access-key' = 'password',
    's3.path-style-access' = 'true'
);
```

```sql
-- カタログ一覧を確認
SHOW CATALOGS;
```

```
+-----------------+
|    catalog name |
+-----------------+
| default_catalog |
| iceberg_catalog |
+-----------------+
```

```sql
-- カタログを使用
USE CATALOG iceberg_catalog;
```

## 基本操作

### データベースとテーブルの作成

```sql
-- データベース作成
CREATE DATABASE IF NOT EXISTS demo;
USE demo;

-- テーブル作成
CREATE TABLE users (
    user_id BIGINT,
    name STRING,
    email STRING,
    score DOUBLE,
    created_at TIMESTAMP(6)
);
```

### データの挿入

```sql
INSERT INTO users VALUES
    (1, 'Alice', 'alice@example.com', 85.5, CURRENT_TIMESTAMP),
    (2, 'Bob', 'bob@example.com', 92.0, CURRENT_TIMESTAMP),
    (3, 'Charlie', 'charlie@example.com', 78.5, CURRENT_TIMESTAMP);
```

INSERTはFlinkジョブとして実行されます。Flink Web UI（http://localhost:8081）の「Running Jobs」でジョブの状態を確認できます。

### データの読み取り

```sql
SELECT * FROM users;
```

```
+---------+---------+---------------------+-------+-------------------------+
| user_id |    name |               email | score |              created_at |
+---------+---------+---------------------+-------+-------------------------+
|       1 |   Alice |   alice@example.com |  85.5 | 2025-12-28 07:48:43.849 |
|       2 |     Bob |     bob@example.com |  92.0 | 2025-12-28 07:48:43.856 |
|       3 | Charlie | charlie@example.com |  78.5 | 2025-12-28 07:48:43.856 |
+---------+---------+---------------------+-------+-------------------------+
```

### 集計クエリ

```sql
SELECT 
    COUNT(*) as total_users,
    AVG(score) as avg_score,
    MAX(score) as max_score,
    MIN(score) as min_score
FROM users;
```

## メタデータテーブル

Icebergのメタデータテーブルにアクセスして、テーブルの内部情報を確認できます。

:::message
ソートを使用する場合は、バッチモードに切り替える必要があります。

```sql
SET 'execution.runtime-mode' = 'batch';
```
:::

### スナップショット一覧

```sql
SELECT snapshot_id, committed_at, operation 
FROM iceberg_catalog.demo.`users$snapshots`
ORDER BY committed_at;
```

### 履歴

```sql
SELECT * FROM iceberg_catalog.demo.`users$history`;
```

### 主なメタデータテーブル

| テーブル | 内容 |
|---------|------|
| `$snapshots` | スナップショット一覧 |
| `$history` | テーブル変更履歴 |
| `$files` | データファイル一覧 |
| `$manifests` | マニフェストファイル一覧 |

## タイムトラベルクエリ

Icebergはスナップショットベースのため、過去の時点のデータにアクセスできます。

まず、追加のデータを挿入してスナップショットを増やします。

```sql
-- 4件目のデータを追加
INSERT INTO users VALUES
    (4, 'David', 'david@example.com', 88.0, CURRENT_TIMESTAMP);

-- 現在のデータを確認（4件）
SELECT * FROM users;
```

```sql
-- スナップショットIDを確認
SELECT snapshot_id, committed_at 
FROM iceberg_catalog.demo.`users$snapshots`
ORDER BY committed_at;
```

最初のスナップショットID（3件追加時点）を指定して、過去のデータを参照します。

```sql
-- スナップショットIDを指定して過去データを参照
SELECT * FROM iceberg_catalog.demo.users 
    /*+ OPTIONS('snapshot-id'='最初のスナップショットID') */;
```

最初のスナップショット時点では3件のみ取得でき、Davidはまだ追加されていないことが確認できます。

:::message
Flink SQLでは`FOR SYSTEM_TIME AS OF`構文はサポートされていないため、テーブルヒント（`/*+ OPTIONS(...) */`）でスナップショットIDを指定します。
:::

## UPSERTモード

PRIMARY KEYを指定し、`write.upsert.enabled`を有効にすると、同じキーのINSERTが更新として扱われます。

```sql
-- UPSERT対応テーブル作成
CREATE TABLE users_upsert (
    user_id BIGINT,
    name STRING,
    email STRING,
    score DOUBLE,
    updated_at TIMESTAMP(6),
    PRIMARY KEY (user_id) NOT ENFORCED
) WITH (
    'format-version' = '2',
    'write.upsert.enabled' = 'true'
);
```

| 設定 | 役割 |
|------|------|
| `PRIMARY KEY ... NOT ENFORCED` | キーとなるカラムを指定 |
| `format-version = 2` | Iceberg v2形式（行レベルの更新をサポート） |
| `write.upsert.enabled = true` | UPSERTモードを有効化 |

```sql
-- 初期データ
INSERT INTO users_upsert VALUES
    (1, 'Alice', 'alice@example.com', 85.5, CURRENT_TIMESTAMP);

SELECT * FROM users_upsert;
```

```
+---------+-------+-------------------+-------+-------------------------+
| user_id |  name |             email | score |              updated_at |
+---------+-------+-------------------+-------+-------------------------+
|       1 | Alice | alice@example.com |  85.5 | 2025-12-28 08:03:45.152 |
+---------+-------+-------------------+-------+-------------------------+
```

```sql
-- 同じuser_idで更新（UPSERT）
INSERT INTO users_upsert VALUES
    (1, 'Alice', 'alice_updated@example.com', 90.0, CURRENT_TIMESTAMP);

SELECT * FROM users_upsert;
```

```
+---------+-------+---------------------------+-------+-------------------------+
| user_id |  name |                     email | score |              updated_at |
+---------+-------+---------------------------+-------+-------------------------+
|       1 | Alice | alice_updated@example.com |  90.0 | 2025-12-28 08:04:23.560 |
+---------+-------+---------------------------+-------+-------------------------+
```

2件にならず、1件のまま更新されています。これはストリーミング処理で特に便利で、同じキーのイベントが複数回来ても最新の状態だけを保持できます。

## まとめ

本記事では、Docker ComposeでFlink + Iceberg環境を構築し、Flink SQLでの基本操作を体験しました。

### 確認できたこと

| 項目 | 状態 |
|------|------|
| 環境構築（Flink + Iceberg + MinIO） | ✅ |
| カタログ接続（REST Catalog） | ✅ |
| テーブル作成・データ挿入 | ✅ |
| 集計クエリ | ✅ |
| メタデータテーブル | ✅ |
| タイムトラベルクエリ | ✅ |
| UPSERTモード | ✅ |

### 今回の記事では扱わなかったこと

Flinkの本領は**ストリーミング処理**にあります。今回はバッチ的な操作が中心でしたが、以下のようなユースケースでFlinkの強みが発揮されます。

- Kafkaからのリアルタイムデータ取り込み
- ウィンドウ処理（5分ごとの集計など）
- 継続的なデータパイプライン

これらについては、別の記事で扱えればと思います。

## 参考資料

- [Apache Flink公式](https://flink.apache.org/)
- [Flink Iceberg Connector](https://iceberg.apache.org/docs/latest/flink/)
- [Apache Iceberg公式](https://iceberg.apache.org/)
