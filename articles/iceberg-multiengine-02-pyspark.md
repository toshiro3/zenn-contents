---
title: "PySparkとPyIcebergでIcebergテーブルを相互運用する"
emoji: "🔄"
type: "tech"
topics: ["ApacheIceberg", "PySpark", "PyIceberg", "DataEngineering", "Docker"]
published: false
---

## はじめに

本記事は「Apache Iceberg マルチエンジン実践ガイド」の第2回です。

| 回 | 内容 |
|----|------|
| [第1回](https://zenn.dev/toshiro3/articles/iceberg-multiengine-01-setup) | 環境構築・PyIcebergでの基本操作 |
| **第2回（本記事）** | PySparkとの相互運用 |
| 第3回 | TrinoでのSQL分析 |

前回構築した環境を使って、**PySparkとPyIcebergで同じIcebergテーブルにアクセス**し、マルチエンジンでの相互運用性を検証します。

### 検証ポイント

1. PyIcebergで作成したテーブルをPySparkで読み取れるか
2. PySparkで書き込んだデータをPyIcebergで読み取れるか
3. スキーマ進化の共有
4. スナップショット履歴の確認

## 環境の準備

第1回で構築した環境を起動します。

```bash
docker compose up -d
```

:::message
REST CatalogはデフォルトでSQLiteのメモリモードで動作するため、コンテナを再起動するとカタログ情報がリセットされます。第1回の操作から続ける場合も、テーブルの再作成が必要です。
:::

まず、PyIcebergでテーブルを作成しておきます（第1回の内容）。

```python
from pyiceberg.catalog import load_catalog
from pyiceberg.schema import Schema
from pyiceberg.types import StringType, LongType, DoubleType, NestedField
import pyarrow as pa
import os

# カタログ接続
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

# ネームスペース作成
try:
    catalog.create_namespace("demo")
except:
    pass

# スキーマ定義
schema = Schema(
    NestedField(1, "user_id", LongType(), required=True),
    NestedField(2, "name", StringType(), required=True),
    NestedField(3, "email", StringType(), required=False),
    NestedField(4, "score", DoubleType(), required=False),
)

# テーブル作成
try:
    table = catalog.create_table("demo.users", schema=schema)
except:
    table = catalog.load_table("demo.users")

# 初期データ追加
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
print("PyIcebergでテーブルを作成し、3件のデータを追加しました")
```

## PySparkセッションの作成

PySparkからIcebergテーブルにアクセスするためのセッションを作成します。

```python
from pyspark.sql import SparkSession
import os

spark = SparkSession.builder \
    .appName("IcebergMultiEngine") \
    .config("spark.sql.extensions", "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions") \
    .config("spark.sql.catalog.rest", "org.apache.iceberg.spark.SparkCatalog") \
    .config("spark.sql.catalog.rest.type", "rest") \
    .config("spark.sql.catalog.rest.uri", os.environ.get("CATALOG_URI", "http://rest-catalog:8181")) \
    .config("spark.sql.catalog.rest.io-impl", "org.apache.iceberg.aws.s3.S3FileIO") \
    .config("spark.sql.catalog.rest.s3.endpoint", os.environ.get("S3_ENDPOINT", "http://minio:9000")) \
    .config("spark.sql.catalog.rest.s3.access-key-id", os.environ.get("AWS_ACCESS_KEY_ID", "admin")) \
    .config("spark.sql.catalog.rest.s3.secret-access-key", os.environ.get("AWS_SECRET_ACCESS_KEY", "password")) \
    .config("spark.sql.catalog.rest.s3.path-style-access", "true") \
    .config("spark.sql.defaultCatalog", "rest") \
    .config("spark.driver.memory", "1g") \
    .getOrCreate()

print(f"Spark version: {spark.version}")
```

### 設定のポイント

| 設定 | 説明 |
|------|------|
| `spark.sql.extensions` | Iceberg用のSpark拡張を有効化 |
| `spark.sql.catalog.rest` | `rest`という名前でカタログを定義 |
| `spark.sql.catalog.rest.type` | REST Catalog方式を指定 |
| `spark.sql.catalog.rest.uri` | REST CatalogのエンドポイントURL |

## 相互運用検証①: PyIceberg → PySpark

PyIcebergで作成したテーブルをPySparkで読み取ります。

### ネームスペース確認

```python
spark.sql("SHOW NAMESPACES").show()
```

```
+---------+
|namespace|
+---------+
|     demo|
+---------+
```

### テーブル確認

```python
spark.sql("SHOW TABLES IN demo").show()
```

```
+---------+---------+-----------+
|namespace|tableName|isTemporary|
+---------+---------+-----------+
|     demo|    users|      false|
+---------+---------+-----------+
```

### データ読み取り

```python
df_spark = spark.sql("SELECT * FROM rest.demo.users")
df_spark.show()
```

```
+-------+-------+-----------------+-----+
|user_id|   name|            email|score|
+-------+-------+-----------------+-----+
|      1|  Alice|alice@example.com| 85.5|
|      2|    Bob|  bob@example.com| 92.0|
|      3|Charlie|             NULL| 78.5|
+-------+-------+-----------------+-----+
```

PyIcebergで作成・追加したデータが、PySparkから正しく読み取れました。

## 相互運用検証②: PySpark → PyIceberg

今度は逆に、PySparkでデータを追加し、PyIcebergで読み取ります。

### PySparkでデータ追加

```python
spark.sql("""
    INSERT INTO rest.demo.users VALUES
    (4, 'David', 'david@example.com', 88.0),
    (5, 'Eve', 'eve@example.com', 95.5)
""")
print("PySparkから2件のデータを追加しました")
```

### PySparkで確認

```python
spark.sql("SELECT * FROM rest.demo.users ORDER BY user_id").show()
```

```
+-------+-------+-----------------+-----+
|user_id|   name|            email|score|
+-------+-------+-----------------+-----+
|      1|  Alice|alice@example.com| 85.5|
|      2|    Bob|  bob@example.com| 92.0|
|      3|Charlie|             NULL| 78.5|
|      4|  David|david@example.com| 88.0|
|      5|    Eve|  eve@example.com| 95.5|
+-------+-------+-----------------+-----+
```

### PyIcebergで確認

```python
# テーブルをリフレッシュして最新状態を取得
table.refresh()

df_pyiceberg = table.scan().to_pandas()
df_pyiceberg
```

| user_id | name | email | score |
|---------|------|-------|-------|
| 1 | Alice | alice@example.com | 85.5 |
| 2 | Bob | bob@example.com | 92.0 |
| 3 | Charlie | None | 78.5 |
| 4 | David | david@example.com | 88.0 |
| 5 | Eve | eve@example.com | 95.5 |

PySparkで追加したデータが、PyIcebergから正しく読み取れました。

:::message
`table.refresh()` を呼び出すことで、他のエンジンによる変更を反映します。
:::

## スキーマ進化

PySparkを使ってカラムを追加し、PyIcebergでも認識されるか確認します。

### PySparkでカラム追加

```python
spark.sql("ALTER TABLE rest.demo.users ADD COLUMNS (created_at TIMESTAMP)")
print("カラム 'created_at' を追加しました")
```

### PySparkでスキーマ確認

```python
spark.sql("DESCRIBE rest.demo.users").show()
```

```
+----------+---------+-------+
|  col_name|data_type|comment|
+----------+---------+-------+
|   user_id|   bigint|       |
|      name|   string|       |
|     email|   string|       |
|     score|   double|       |
|created_at|timestamp|       |
+----------+---------+-------+
```

### PyIcebergでスキーマ確認

```python
table.refresh()

print("=== PyIcebergでスキーマを確認 ===")
for field in table.schema().fields:
    print(f"  {field.field_id}: {field.name} ({field.field_type}) required={field.required}")
```

```
=== PyIcebergでスキーマを確認 ===
  1: user_id (long) required=True
  2: name (string) required=True
  3: email (string) required=False
  4: score (double) required=False
  5: created_at (timestamp) required=False
```

PySparkで追加したカラムが、PyIcebergでも認識されています。

## スナップショット履歴の確認

両エンジンからの操作がスナップショットとしてどのように記録されているか確認します。

```python
table.refresh()

for snap in table.metadata.snapshots:
    app_id = snap.summary.get("app-id") or ""
    engine = "PyIceberg" if "pyiceberg" in app_id.lower() else "Spark"
    print(f"Snapshot ID: {snap.snapshot_id}")
    print(f"  Operation: {snap.summary.get('operation')}")
    print(f"  Added Records: {snap.summary.get('added-records', '0')}")
    print(f"  Engine: {engine}")
    print()
```

```
Snapshot ID: 1234567890123456789
  Operation: append
  Added Records: 3
  Engine: PyIceberg

Snapshot ID: 2345678901234567890
  Operation: append
  Added Records: 2
  Engine: Spark
```

スナップショットの `app-id` フィールドから、どのエンジンで操作が行われたか判別できます。

## PySparkとPyIcebergの使い分け

両エンジンにはそれぞれ強みがあります。

| 観点 | PySpark | PyIceberg |
|------|---------|-----------|
| **JVM依存** | 必要 | 不要 |
| **起動時間** | 遅い（数秒〜） | 速い |
| **分散処理** | ✅ 対応 | ❌ 単一プロセス |
| **SQL操作** | ✅ Spark SQL | ❌ なし |
| **メタデータ操作** | △ 基本的な操作 | ✅ 詳細なAPI |
| **適したユースケース** | 大規模ETL、SQL分析 | 軽量な読み書き、メタデータ操作 |

### 使い分けの例

```python
# PyIceberg: メタデータの確認、軽量な読み取り
table = catalog.load_table("demo.users")
snapshots = table.metadata.snapshots
df = table.scan(row_filter="score > 90").to_pandas()

# PySpark: 大規模データ処理、複雑なSQL
spark.sql("""
    SELECT name, AVG(score) as avg_score
    FROM rest.demo.users
    GROUP BY name
    HAVING AVG(score) > 80
""").show()
```

## まとめ

本記事では、PySparkとPyIcebergの相互運用性を検証しました。

✅ **確認できたこと**

| 操作 | PyIceberg → PySpark | PySpark → PyIceberg |
|------|---------------------|---------------------|
| データ読み取り | ✅ | ✅ |
| データ書き込み | ✅ | ✅ |
| スキーマ進化 | ✅ | ✅ |
| スナップショット共有 | ✅ | ✅ |

REST Catalogを介することで、異なるエンジンが同じIcebergテーブルに一貫してアクセスできることが確認できました。

次回は、Trinoを使ったSQL分析と、Icebergのメタデータテーブルを活用した詳細な分析を行います。

---

**前の記事**: [Docker ComposeでApache Icebergマルチエンジン検証環境を構築する](https://zenn.dev/toshiro3/articles/iceberg-multiengine-01-setup)
