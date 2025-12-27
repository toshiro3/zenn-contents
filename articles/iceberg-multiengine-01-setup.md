---
title: "Docker ComposeでApache Icebergマルチエンジン検証環境を構築する"
emoji: "🧊"
type: "tech"
topics: ["ApacheIceberg", "PyIceberg", "Docker", "DataEngineering", "marimo"]
published: true
---

## はじめに

Apache Icebergの大きな強みの一つは、**複数のエンジンから同じテーブルにアクセスできる**ことです。Spark、Trino、PyIcebergなど、用途に応じて最適なエンジンを選択しながら、データの一貫性を保つことができます。

本シリーズでは、Docker Composeを使ってローカルにマルチエンジン検証環境を構築し、実際に複数エンジンでの相互運用を体験します。

### シリーズ構成

| 回 | 内容 |
|----|------|
| **第1回（本記事）** | 環境構築・PyIcebergでの基本操作 |
| 第2回 | PySparkとの相互運用 |
| 第3回 | TrinoでのSQL分析 |

### この記事で構築する環境

```
┌─────────────────────────────────────────────────────────┐
│                   marimo (port: 2718)                    │
│            PyIceberg / PySpark / Trino Client            │
└──────────────────────────┬──────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
┌─────────────────────┐    ┌─────────────────────┐
│   REST Catalog      │    │      Trino          │
│   (port: 8181)      │    │   (port: 8080)      │
└──────────┬──────────┘    └──────────┬──────────┘
           │                          │
           └────────────┬─────────────┘
                        │
                        ▼
           ┌─────────────────────┐
           │       MinIO         │
           │  S3 API: 9000       │
           │  Console: 9001      │
           └─────────────────────┘
```

## REST Catalogとは

Icebergでは、テーブルのメタデータを管理する「カタログ」が必要です。カタログにはいくつかの種類があります。

| カタログ種類 | 特徴 | 備考 |
|-------------|------|------|
| JDBC Catalog | RDBMSでメタデータ管理 | PostgreSQL、MySQL等で本番利用可 |
| REST Catalog | REST API経由でアクセス | エンジン非依存のアクセスが容易 |
| Glue Catalog | AWS Glue Data Catalog | AWSサービスとの統合 |
| Hive Metastore | Hive互換 | 既存Hadoop資産の活用 |

:::message
JDBC CatalogはPyIcebergでは「SQL Catalog」という名前で実装されています。本記事ではIceberg公式ドキュメントに合わせて「JDBC Catalog」と表記します。
:::

**REST Catalog**を使う理由は、**エンジン非依存**だからです。PyIceberg、Spark、Trinoなど異なるエンジンが、同じREST APIを通じてカタログにアクセスできます。

今回は`tabulario/iceberg-rest`イメージを使用します。内部的にはJDBC Catalog（SQLite）をREST APIでラップしています。

:::message
デフォルトではSQLiteの**メモリモード**で動作するため、コンテナを再起動するとカタログ情報がリセットされます。PostgreSQLなどの外部DBを指定することで永続化も可能ですが、本シリーズではチュートリアル用途のためデフォルト設定を使用します。
:::

## 環境構築

本記事で使用する環境はGitHubリポジトリで公開しています。

https://github.com/toshiro3/iceberg-multiengine-lab

```bash
git clone https://github.com/toshiro3/iceberg-multiengine-lab.git
cd iceberg-multiengine-lab
```

:::message
リポジトリには完成版のノートブック（`notebooks/`）が含まれています。記事を読みながら手を動かしたい方はコードをコピーして、すぐに動かしたい方は完成版をお使いください。
:::

### ディレクトリ構成

```
iceberg-multiengine-lab/
├── docker-compose.yml
├── marimo/
│   ├── Dockerfile
│   └── requirements.txt
├── trino/
│   └── catalog/
│       └── iceberg.properties
├── notebooks/
├── minio-data/
└── warehouse/
```

### docker-compose.yml

```yaml
services:
  # ===========================================
  # marimo: メインのノートブック環境
  # PyIceberg + PySpark対応
  # ===========================================
  marimo:
    build:
      context: ./marimo
      dockerfile: Dockerfile
    container_name: marimo
    ports:
      - "2718:2718"
    volumes:
      - ./notebooks:/app/notebooks
      - ./warehouse:/app/warehouse
    environment:
      - AWS_ACCESS_KEY_ID=admin
      - AWS_SECRET_ACCESS_KEY=password
      - AWS_REGION=us-east-1
      - AWS_DEFAULT_REGION=us-east-1
      - CATALOG_URI=http://rest-catalog:8181
      - S3_ENDPOINT=http://minio:9000
    depends_on:
      rest-catalog:
        condition: service_started
      minio-init:
        condition: service_completed_successfully
    networks:
      - iceberg-net
    restart: on-failure

  # ===========================================
  # REST Catalog: Icebergカタログサーバー
  # ===========================================
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
      - iceberg-net
    restart: on-failure

  # ===========================================
  # MinIO: S3互換オブジェクトストレージ
  # ===========================================
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
      - iceberg-net

  # ===========================================
  # MinIO初期設定: warehouseバケット作成
  # ===========================================
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
      echo 'Bucket created successfully';
      exit 0;
      "
    networks:
      - iceberg-net

  # ===========================================
  # Trino: 高速SQLエンジン（第3回で使用）
  # ===========================================
  trino:
    image: trinodb/trino:435
    container_name: trino
    ports:
      - "8080:8080"
    volumes:
      - ./trino/catalog:/etc/trino/catalog
    depends_on:
      rest-catalog:
        condition: service_started
    networks:
      - iceberg-net
    restart: on-failure

networks:
  iceberg-net:
    driver: bridge
```

### marimo/Dockerfile

```dockerfile
FROM python:3.11-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    default-jdk-headless \
    curl \
    procps \
    && rm -rf /var/lib/apt/lists/*

ENV JAVA_HOME=/usr/lib/jvm/default-java
ENV PYSPARK_PYTHON=python3

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# PySparkのjarsにIceberg JARを追加
RUN PYSPARK_JARS=$(python -c "import pyspark; print(pyspark.__path__[0])")/jars && \
    curl -L -o ${PYSPARK_JARS}/iceberg-spark-runtime-3.5_2.12-1.6.1.jar \
    https://repo1.maven.org/maven2/org/apache/iceberg/iceberg-spark-runtime-3.5_2.12/1.6.1/iceberg-spark-runtime-3.5_2.12-1.6.1.jar && \
    curl -L -o ${PYSPARK_JARS}/iceberg-aws-bundle-1.6.1.jar \
    https://repo1.maven.org/maven2/org/apache/iceberg/iceberg-aws-bundle/1.6.1/iceberg-aws-bundle-1.6.1.jar

RUN mkdir -p /app/notebooks /app/warehouse

EXPOSE 2718

CMD ["marimo", "edit", "--host", "0.0.0.0", "-p", "2718", "--no-token", "/app/notebooks"]
```

### marimo/requirements.txt

```
marimo[sql]>=0.10.0
pyiceberg[pyarrow,s3fs]>=0.8.0
pyspark==3.5.3
pandas>=2.0.0
pyarrow>=14.0.0
boto3>=1.34.0
s3fs>=2024.2.0
trino>=0.329.0
plotly>=5.18.0
altair>=5.2.0
```

### trino/catalog/iceberg.properties

```properties
connector.name=iceberg
iceberg.catalog.type=rest
iceberg.rest-catalog.uri=http://rest-catalog:8181
iceberg.rest-catalog.warehouse=s3://warehouse/
fs.native-s3.enabled=true
s3.endpoint=http://minio:9000
s3.path-style-access=true
s3.aws-access-key=admin
s3.aws-secret-key=password
s3.region=us-east-1
```

### 起動

```bash
docker compose up -d
```

起動後、以下のURLでアクセスできます。

| サービス | URL | 用途 |
|---------|-----|------|
| marimo | http://localhost:2718 | ノートブック |
| MinIO Console | http://localhost:9001 | ストレージ管理（admin/password） |
| Trino | http://localhost:8080 | SQLクエリUI（第3回で使用） |

## PyIcebergでの基本操作

marimoでノートブックを開き、PyIcebergの基本操作を試します。新規作成する場合は以下のコードを入力し、完成版を使う場合は `01_pyiceberg_intro.py` を開いてください。

### カタログに接続

```python
from pyiceberg.catalog import load_catalog
import os

catalog = load_catalog(
    "rest",
    **{
        "type": "rest",
        "uri": os.environ.get("CATALOG_URI", "http://rest-catalog:8181"),
        "s3.endpoint": os.environ.get("S3_ENDPOINT", "http://minio:9000"),
        "s3.access-key-id": os.environ.get("AWS_ACCESS_KEY_ID", "admin"),
        "s3.secret-access-key": os.environ.get("AWS_SECRET_ACCESS_KEY", "password"),
        "s3.region": "us-east-1",
    }
)

print(f"カタログ接続成功: {catalog}")
```

### ネームスペースの作成

```python
# ネームスペースを作成
try:
    catalog.create_namespace("demo")
    print("ネームスペース 'demo' を作成しました")
except Exception as e:
    print(f"ネームスペース 'demo' は既に存在します")

print(f"ネームスペース一覧: {catalog.list_namespaces()}")
```

### テーブルの作成

```python
from pyiceberg.schema import Schema
from pyiceberg.types import StringType, LongType, DoubleType, NestedField

schema = Schema(
    NestedField(1, "user_id", LongType(), required=True),
    NestedField(2, "name", StringType(), required=True),
    NestedField(3, "email", StringType(), required=False),
    NestedField(4, "score", DoubleType(), required=False),
)

table_name = "demo.users"
try:
    table = catalog.create_table(table_name, schema=schema)
    print(f"テーブル '{table_name}' を作成しました")
except Exception as e:
    print(f"テーブル '{table_name}' は既に存在します。ロードします。")
    table = catalog.load_table(table_name)

print(table)
```

### データの追加

```python
import pyarrow as pa

arrow_schema = pa.schema([
    pa.field("user_id", pa.int64(), nullable=False),
    pa.field("name", pa.string(), nullable=False),
    pa.field("email", pa.string(), nullable=True),
    pa.field("score", pa.float64(), nullable=True),
])

data = pa.table({
    "user_id": [1, 2, 3],
    "name": ["Alice", "Bob", "Charlie"],
    "email": ["alice@example.com", "bob@example.com", None],
    "score": [85.5, 92.0, 78.5],
}, schema=arrow_schema)

table.append(data)
print("3件のデータを追加しました")
```

### データの読み取り

```python
table.refresh()
df = table.scan().to_pandas()
df
```

| user_id | name | email | score |
|---------|------|-------|-------|
| 1 | Alice | alice@example.com | 85.5 |
| 2 | Bob | bob@example.com | 92.0 |
| 3 | Charlie | None | 78.5 |

### スナップショットの確認

```python
for snap in table.metadata.snapshots:
    print(f"Snapshot ID: {snap.snapshot_id}")
    print(f"  Operation: {snap.summary.get('operation')}")
    print(f"  Added Records: {snap.summary.get('added-records')}")
```

## MinIOでファイル構造を確認

MinIO Console（http://localhost:9001）にアクセスすると、Icebergが作成したファイルを確認できます。

```
warehouse/
└── demo/
    └── users/
        ├── metadata/
        │   ├── 00000-xxxxxxxx.metadata.json
        │   └── snap-xxxxxxxx.avro
        └── data/
            └── 00000-0-xxxxxxxx.parquet
```

| ディレクトリ | 内容 |
|-------------|------|
| `metadata/` | テーブル定義、スナップショット情報 |
| `data/` | 実データ（Parquet形式） |

Icebergの特徴として、メタデータとデータが分離して管理されています。これにより、スキーマ進化やタイムトラベルが可能になります。

## まとめ

本記事では、Docker Composeを使ってApache Icebergのマルチエンジン検証環境を構築しました。

- **REST Catalog**: エンジン非依存のカタログアクセス
- **MinIO**: S3互換ストレージ（本番環境に近い構成）
- **marimo**: リアクティブなPythonノートブック

次回は、この環境を使ってPySparkとPyIcebergの相互運用を検証します。
