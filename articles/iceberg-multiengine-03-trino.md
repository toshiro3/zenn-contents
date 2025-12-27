---
title: "TrinoでIcebergテーブルをSQL分析する"
emoji: "🔍"
type: "tech"
topics: ["ApacheIceberg", "Trino", "SQL", "DataEngineering", "Docker"]
published: false
---

## はじめに

本記事は「Apache Iceberg マルチエンジン実践ガイド」の最終回です。

| 回 | 内容 |
|----|------|
| [第1回](https://zenn.dev/toshiro3/articles/iceberg-multiengine-01-setup) | 環境構築・PyIcebergでの基本操作 |
| [第2回](https://zenn.dev/toshiro3/articles/iceberg-multiengine-02-pyspark) | PySparkとの相互運用 |
| **第3回（本記事）** | TrinoでのSQL分析 |

本記事では、**Trino**を使ってIcebergテーブルにSQLでアクセスし、Icebergのメタデータテーブルを活用した分析を行います。

### Trinoとは

Trinoは、分散SQLクエリエンジンです（旧PrestoSQL）。

| 特徴 | 説明 |
|------|------|
| **高速** | メモリベースの分散処理 |
| **多様なデータソース** | Iceberg、Delta Lake、Hive、RDBMS等 |
| **標準SQL** | ANSI SQL準拠 |
| **BIツール連携** | JDBC/ODBC対応 |

大規模データの分析クエリに適したエンジンで、元々Facebookが開発し、現在はNetflixやLinkedInなどでも採用されています（公式サイトより）。GitHubでは12,000以上のスターがあり、開発も活発です。

## Trinoの接続設定

Trinoの設定は `trino/catalog/iceberg.properties` で行います。

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

| 設定 | 説明 |
|------|------|
| `connector.name=iceberg` | Icebergコネクタを使用 |
| `iceberg.catalog.type=rest` | REST Catalog方式 |
| `iceberg.rest-catalog.uri` | REST Catalogのエンドポイント |

ファイル名 `iceberg.properties` がそのままカタログ名 `iceberg` になります。

## Pythonからの接続

marimoノートブックからTrinoに接続します。

```python
from trino.dbapi import connect
import pandas as pd

conn = connect(
    host="trino",
    port=8080,
    user="marimo",
    catalog="iceberg",
    schema="demo",
)

print("Trino接続成功")
```

### ヘルパー関数

クエリ実行を簡単にするヘルパー関数を定義します。

```python
def run_query(sql):
    """SQLを実行してDataFrameを返す"""
    cursor = conn.cursor()
    cursor.execute(sql)
    columns = [desc[0] for desc in cursor.description]
    return pd.DataFrame(cursor.fetchall(), columns=columns)
```

## 基本SQL操作

### カタログとスキーマの確認

```python
run_query("SHOW CATALOGS")
```

| Catalog |
|---------|
| iceberg |
| system |

```python
run_query("SHOW SCHEMAS IN iceberg")
```

| Schema |
|--------|
| demo |
| information_schema |

### テーブル確認

```python
run_query("SHOW TABLES IN iceberg.demo")
```

| Table |
|-------|
| users |

### データ読み取り

```python
run_query("SELECT * FROM iceberg.demo.users ORDER BY user_id")
```

| user_id | name | email | score | created_at |
|---------|------|-------|-------|------------|
| 1 | Alice | alice@example.com | 85.5 | None |
| 2 | Bob | bob@example.com | 92.0 | None |
| 3 | Charlie | None | 78.5 | None |
| 4 | David | david@example.com | 88.0 | None |
| 5 | Eve | eve@example.com | 95.5 | None |

PyIcebergとPySparkで追加したデータが、Trinoからも正しく読み取れています。

### 集計クエリ

```python
run_query("""
    SELECT 
        COUNT(*) as total_users,
        AVG(score) as avg_score,
        MAX(score) as max_score,
        MIN(score) as min_score
    FROM iceberg.demo.users
""")
```

| total_users | avg_score | max_score | min_score |
|-------------|-----------|-----------|-----------|
| 5 | 87.9 | 95.5 | 78.5 |

## Trinoからデータを追加

Trinoからもデータを追加できます。

```python
cursor = conn.cursor()
cursor.execute("""
    INSERT INTO iceberg.demo.users (user_id, name, email, score)
    VALUES (6, 'Frank', 'frank@example.com', 82.0)
""")
print("Trinoから1件のデータを追加しました")
```

```python
run_query("SELECT * FROM iceberg.demo.users ORDER BY user_id")
```

| user_id | name | email | score | created_at |
|---------|------|-------|-------|------------|
| 1 | Alice | alice@example.com | 85.5 | None |
| 2 | Bob | bob@example.com | 92.0 | None |
| 3 | Charlie | None | 78.5 | None |
| 4 | David | david@example.com | 88.0 | None |
| 5 | Eve | eve@example.com | 95.5 | None |
| 6 | Frank | frank@example.com | 82.0 | None |

これで、3つのエンジン（PyIceberg、PySpark、Trino）すべてからデータを追加しました。

## メタデータテーブルの活用

Icebergの大きな特徴として、**メタデータテーブル**があります。テーブル名に `$` サフィックスを付けることで、メタデータにSQLでアクセスできます。

### スナップショット一覧 (`$snapshots`)

```python
run_query("""
    SELECT 
        snapshot_id,
        committed_at,
        operation,
        summary
    FROM iceberg.demo."users$snapshots"
    ORDER BY committed_at DESC
""")
```

| snapshot_id | committed_at | operation | summary |
|-------------|--------------|-----------|---------|
| 345... | 2024-12-27 02:45:00 | append | {added-records=1, ...} |
| 234... | 2024-12-27 02:30:00 | append | {added-records=2, ...} |
| 123... | 2024-12-27 02:15:00 | append | {added-records=3, ...} |

各操作がスナップショットとして記録されていることがわかります。

### データファイル一覧 (`$files`)

```python
run_query("""
    SELECT 
        file_path,
        file_format,
        record_count,
        file_size_in_bytes
    FROM iceberg.demo."users$files"
""")
```

| file_path | file_format | record_count | file_size_in_bytes |
|-----------|-------------|--------------|-------------------|
| s3://warehouse/demo/users/data/00000-0-xxx.parquet | PARQUET | 3 | 1234 |
| s3://warehouse/demo/users/data/00001-0-xxx.parquet | PARQUET | 2 | 1100 |
| s3://warehouse/demo/users/data/00002-0-xxx.parquet | PARQUET | 1 | 900 |

実際のデータファイルの情報を確認できます。

### 履歴 (`$history`)

```python
run_query("""
    SELECT *
    FROM iceberg.demo."users$history"
    ORDER BY made_current_at DESC
""")
```

テーブルの変更履歴を時系列で確認できます。

### マニフェスト (`$manifests`)

```python
run_query("""
    SELECT 
        path,
        added_snapshot_id,
        added_data_files_count,
        added_rows_count
    FROM iceberg.demo."users$manifests"
""")
```

マニフェストファイル（データファイルのインデックス）の情報を確認できます。

### 主なメタデータテーブル一覧

| テーブル | 内容 |
|---------|------|
| `$snapshots` | スナップショット一覧 |
| `$history` | テーブル変更履歴 |
| `$files` | データファイル一覧 |
| `$manifests` | マニフェストファイル一覧 |
| `$partitions` | パーティション情報 |
| `$refs` | ブランチ・タグ情報 |

## タイムトラベルクエリ

Icebergはスナップショットベースのため、過去の時点のデータにアクセスできます。

### スナップショットIDを指定

```python
# まずスナップショットIDを取得
snapshots = run_query("""
    SELECT snapshot_id, committed_at 
    FROM iceberg.demo."users$snapshots"
    ORDER BY committed_at ASC
    LIMIT 1
""")

first_snapshot_id = snapshots['snapshot_id'].iloc[0]
print(f"最初のスナップショットID: {first_snapshot_id}")
```

```python
# 過去のスナップショットを参照
run_query(f"""
    SELECT * 
    FROM iceberg.demo.users 
    FOR VERSION AS OF {first_snapshot_id}
""")
```

| user_id | name | email | score |
|---------|------|-------|-------|
| 1 | Alice | alice@example.com | 85.5 |
| 2 | Bob | bob@example.com | 92.0 |
| 3 | Charlie | None | 78.5 |

最初のスナップショット時点（3件のみ）のデータが取得できました。

### タイムスタンプを指定

```python
# 既に取得済みのsnapshotsから時刻を取得
first_snapshot_time = snapshots['committed_at'].iloc[0]
print(f"最初のスナップショット時刻: {first_snapshot_time}")
```

```python
# 過去の時刻を指定して参照
run_query(f"""
    SELECT * 
    FROM iceberg.demo.users 
    FOR TIMESTAMP AS OF TIMESTAMP '{first_snapshot_time}'
""")
```

指定した時刻時点のデータを取得できます。

## 3エンジン比較まとめ

本シリーズで検証した3つのエンジンを比較します。

| 観点 | PyIceberg | PySpark | Trino |
|------|-----------|---------|-------|
| **言語** | Python | Python/Scala/Java | SQL |
| **JVM依存** | ❌ 不要 | ✅ 必要 | ✅ 必要 |
| **分散処理** | ❌ | ✅ | ✅ |
| **起動速度** | 速い | 遅い | 中程度 |
| **SQL対応** | ❌ | ✅ Spark SQL | ✅ ANSI SQL |
| **メタデータ操作** | ✅ 詳細 | △ 基本的 | ✅ SQLで可能 |
| **BIツール連携** | ❌ | △ | ✅ JDBC/ODBC |

### ユースケース別の使い分け

| ユースケース | 推奨エンジン |
|-------------|-------------|
| 軽量なデータ読み書き | PyIceberg |
| メタデータの詳細操作 | PyIceberg |
| 大規模ETL処理 | PySpark |
| 複雑なデータ変換 | PySpark |
| 分析クエリ | Trino |
| BIツール連携 | Trino |
| アドホック分析 | Trino |

## 本番環境への展望

本シリーズではローカル環境で検証しましたが、本番環境では以下のような構成がとりやすそうです。

### AWS環境

| コンポーネント | 本シリーズ | AWS本番 |
|---------------|-----------|---------|
| カタログ | REST Catalog（SQLite） | Glue Data Catalog |
| ストレージ | MinIO | S3 / S3 Tables |
| コンピュート | Docker | EMR / Athena / Glue |

### GCP環境

| コンポーネント | 本シリーズ | GCP本番 |
|---------------|-----------|---------|
| カタログ | REST Catalog（SQLite） | BigLake Metastore |
| ストレージ | MinIO | GCS |
| コンピュート | Docker | Dataproc / BigQuery |

:::message
REST CatalogはAPI仕様であり、Glue Data CatalogやBigLake MetastoreもREST Catalog API経由でアクセスできます。本シリーズで学んだREST Catalogへの接続方法は、本番環境でも活かせそうです。
:::

## まとめ

本シリーズでは、Docker Composeを使ってApache Icebergのマルチエンジン検証環境を構築し、3つのエンジンでの相互運用性を確認しました。

### 確認できたこと

✅ REST Catalogを介した複数エンジンからの一貫したアクセス
✅ エンジン間でのデータ・スキーマ・スナップショットの共有
✅ Icebergメタデータテーブルを活用した詳細分析
✅ タイムトラベルクエリによる過去データへのアクセス

### シリーズで構築した環境

```
┌─────────────────────────────────────────────────────────┐
│                      marimo                              │
│         PyIceberg  /  PySpark  /  Trino Client          │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
                  ┌───────────────┐
                  │  REST Catalog │  ← 統一されたアクセスポイント
                  └───────┬───────┘
                          │
                          ▼
                  ┌───────────────┐
                  │     MinIO     │  ← S3互換ストレージ
                  └───────────────┘
```

Apache Icebergのマルチエンジン対応は、データレイクハウスアーキテクチャの大きな強みと言えそうです。本シリーズが、Icebergを活用したデータ基盤構築の参考になれば幸いです。

---

**前の記事**: [PySparkとPyIcebergでIcebergテーブルを相互運用する](https://zenn.dev/toshiro3/articles/iceberg-multiengine-02-pyspark)

## 参考資料

- [Apache Iceberg公式ドキュメント](https://iceberg.apache.org/)
- [PyIceberg](https://py.iceberg.apache.org/)
- [Trino Iceberg Connector](https://trino.io/docs/current/connector/iceberg.html)
- [marimo](https://marimo.io/)
