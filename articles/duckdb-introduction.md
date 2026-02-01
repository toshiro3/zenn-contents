---
title: "【入門】DuckDBとは？MySQL/PostgreSQLとの違いを手を動かして理解する"
emoji: "🦆"
type: "tech"
topics: ["duckdb", "python", "database", "データエンジニアリング", "分析"]
published: true
---

## 🚀 忙しい人のためのDuckDB 3行要約

- **「分析版SQLite」**: `pip install duckdb` するだけで使えるサーバーレスなDBでありながら、大量データの集計に特化した圧倒的な性能を持つ。

- **インポート不要のクエリ**: CSVやParquetファイルをDBに取り込む手間なく、そのままSQLで分析・集計が可能。

- **データ分析の万能ナイフ**: ログ集計、ETL処理（データ変換）、大規模データの探索など、エンジニアの「手元の作業」を劇的に速くする。

## ⏱️ クイックスタート（コピペで即座に集計）

Python環境があれば、DuckDBだけで今すぐ実行できるサンプルです。

```python
import duckdb

# 1. 100万件のテストデータを一瞬で生成して集計
# サーバー不要で「100万件」を0.1秒以下で扱えるのがDuckDBの魅力です
result = duckdb.sql("""
    SELECT 
        (i % 5) AS group_id, 
        COUNT(*) AS total_count,
        AVG(random()) AS avg_value
    FROM generate_series(1, 1000000) AS t(i)
    GROUP BY (i % 5)
    ORDER BY group_id
""")
print(result)

# 2. 結果をそのままCSVとして書き出し（1行でファイル出力）
duckdb.sql("COPY (SELECT 42 AS answer) TO 'quick_start.csv' (FORMAT CSV)")
print("\n'quick_start.csv' を作成しました")
```

Pandasや外部ファイルとの連携は、この記事の後半で詳しく解説します！

## はじめに

データエンジニアリングの技術を調べていると、最近よく目にするようになった「DuckDB」。Dagster + dbt の連携を検証していた際に候補として出てきて興味を持ちました。

「SQLiteのようなもの？」「何がすごいの？」「MySQLと何が違うの？」

そんな疑問を解消するため、実際に手を動かして検証してみました。この記事では、DuckDBの基本的な使い方から、既存のRDB（MySQL/PostgreSQL）との違い、どういった場面で使えるかまでを整理します。

### この記事で学べること

- DuckDBの基本概念（OLAP、列指向、組み込み型）
- Python環境でのセットアップ方法
- CSV/Parquetファイルの直接クエリ
- MySQLなど既存RDBとの違い
- DuckDBならではの便利な機能
- 具体的なユースケース

### 検証環境

- macOS
- Python 3.12.12
- DuckDB 1.4.4
- uv（パッケージ管理）

## DuckDBとは

DuckDBを理解するために、まず3つのキーワードを押さえておきましょう。

### OLAP（Online Analytical Processing）

データベースは大きく2種類に分けられます。

| 種類 | 特徴 | 代表例 |
|------|------|--------|
| OLTP | トランザクション処理向け、1行単位の読み書きが得意 | MySQL, PostgreSQL |
| OLAP | 分析処理向け、大量データの集計が得意 | DuckDB, BigQuery, Redshift |

DuckDBはOLAP型のデータベースです。Webアプリのバックエンドのような「1件ずつデータを出し入れする」用途ではなく、「大量のデータを集計・分析する」用途に特化しています。

### 列指向（Columnar）

従来のRDB（MySQL等）は「行指向」でデータを格納します。

```
# 行指向（MySQL等）
Row1: [id=1, name="田中", age=28, dept="Sales"]
Row2: [id=2, name="鈴木", age=35, dept="Marketing"]
Row3: [id=3, name="佐藤", age=22, dept="Sales"]
```

一方、DuckDBは「列指向」でデータを格納します。

```
# 列指向（DuckDB）
id列:   [1, 2, 3]
name列: ["田中", "鈴木", "佐藤"]
age列:  [28, 35, 22]
dept列: ["Sales", "Marketing", "Sales"]
```

この違いが重要なのは、分析クエリの特性にあります。

```sql
-- 平均年齢を求めるクエリ
SELECT AVG(age) FROM users;
```

このクエリを実行する場合、行指向では全行のデータを読み込む必要がありますが、列指向なら`age`列だけを読み込めば済みます。大量データの集計では、この差が大きなパフォーマンス差を生みます。

### 組み込み型（Embedded）

DuckDBはSQLiteと同様の「組み込み型」データベースです。

| 特徴 | 組み込み型（DuckDB, SQLite） | サーバー型（MySQL, PostgreSQL） |
|------|------------------------------|--------------------------------|
| プロセス | アプリケーションに組み込み | 別プロセスで起動 |
| セットアップ | `pip install`で即使える | サーバーのインストール・設定が必要 |
| 接続 | ファイルパスを指定 | ホスト・ポート・認証情報が必要 |
| 運用 | 管理不要 | 監視・バックアップ等が必要 |

手軽に使えるのに、分析性能は本格的。これがDuckDBの魅力です。

## 環境構築

それでは実際にDuckDBを動かしてみましょう。

### プロジェクトのセットアップ

今回はuvを使ってPython環境を構築します。

```bash
# プロジェクトディレクトリ作成
mkdir duckdb-tutorial
cd duckdb-tutorial

# uvでプロジェクト初期化（Python 3.12指定）
uv init --python 3.12

# DuckDBインストール
uv add duckdb
```

バージョン確認します。

```bash
uv run python -c "import duckdb; print(duckdb.__version__)"
```

```
1.4.4
```

これだけで準備完了です。MySQLのようにサーバーを起動する必要はありません。

:::message
**Pythonバージョンについて**

DuckDBはPython 3.9以上をサポートしています。本記事ではPython 3.12で検証しました。
:::

## 基本操作

まずはシンプルなクエリから試してみます。

```python:01_basic.py
import duckdb

# 1. シンプルなクエリ実行
print("=== 1. シンプルなクエリ ===")
result = duckdb.sql("SELECT 42 AS answer, 'Hello DuckDB' AS message")
print(result)

# 2. テーブル作成とデータ挿入（インメモリ）
print("\n=== 2. テーブル作成 ===")
duckdb.sql("""
    CREATE TABLE users (
        id INTEGER,
        name VARCHAR,
        age INTEGER
    )
""")

duckdb.sql("""
    INSERT INTO users VALUES
    (1, '田中', 28),
    (2, '鈴木', 35),
    (3, '佐藤', 22)
""")

result = duckdb.sql("SELECT * FROM users")
print(result)

# 3. 集計クエリ
print("\n=== 3. 集計クエリ ===")
result = duckdb.sql("SELECT AVG(age) AS avg_age, MAX(age) AS max_age FROM users")
print(result)

# 4. 結果をPythonで受け取る
print("\n=== 4. Pythonリストとして取得 ===")
rows = duckdb.sql("SELECT * FROM users").fetchall()
print(rows)
```

```bash
uv run python 01_basic.py
```

実行結果:

```
=== 1. シンプルなクエリ ===
┌────────┬──────────────┐
│ answer │   message    │
│ int32  │   varchar    │
├────────┼──────────────┤
│     42 │ Hello DuckDB │
└────────┴──────────────┘


=== 2. テーブル作成 ===
┌───────┬─────────┬───────┐
│  id   │  name   │  age  │
│ int32 │ varchar │ int32 │
├───────┼─────────┼───────┤
│     1 │ 田中    │    28 │
│     2 │ 鈴木    │    35 │
│     3 │ 佐藤    │    22 │
└───────┴─────────┴───────┘


=== 3. 集計クエリ ===
┌────────────────────┬─────────┐
│      avg_age       │ max_age │
│       double       │  int32  │
├────────────────────┼─────────┤
│ 28.333333333333332 │      35 │
└────────────────────┴─────────┘


=== 4. Pythonリストとして取得 ===
[(1, '田中', 28), (2, '鈴木', 35), (3, '佐藤', 22)]
```

SQLの文法は標準的なので、MySQLやPostgreSQLを使ったことがあればすぐに馴染めます。出力が整形されたテーブル形式で表示されるのも特徴的ですね。

## CSVファイルの直接読み込み

ここからがDuckDBの真骨頂です。MySQLでCSVを読み込むには`LOAD DATA`文を使ったり、外部ツールでインポートしたりする必要がありますが、DuckDBならファイルパスを直接指定するだけでクエリできます。

### サンプルデータの作成

まず、検証用のCSVファイルを作成します。

```python:02_create_csv.py
import csv
import random

# 売上データを生成（10万件）
products = ['商品A', '商品B', '商品C', '商品D', '商品E']
regions = ['東京', '大阪', '名古屋', '福岡', '札幌']

with open('sales.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.writer(f)
    writer.writerow(['id', 'product', 'region', 'quantity', 'price'])
    
    for i in range(100000):
        writer.writerow([
            i + 1,
            random.choice(products),
            random.choice(regions),
            random.randint(1, 100),
            random.randint(100, 10000)
        ])

print("sales.csv を作成しました（10万件）")
```

### CSVを直接クエリ

```python:03_query_csv.py
import duckdb
import time

# 1. CSVファイルを直接SELECT（インポート不要！）
print("=== 1. CSVを直接クエリ ===")
start = time.time()
result = duckdb.sql("SELECT * FROM 'sales.csv' LIMIT 5")
print(result)
print(f"実行時間: {time.time() - start:.3f}秒")

# 2. 集計クエリ（10万件を集計）
print("\n=== 2. 地域別・商品別の売上集計 ===")
start = time.time()
result = duckdb.sql("""
    SELECT 
        region,
        product,
        COUNT(*) AS count,
        SUM(quantity * price) AS total_sales
    FROM 'sales.csv'
    GROUP BY region, product
    ORDER BY total_sales DESC
    LIMIT 10
""")
print(result)
print(f"実行時間: {time.time() - start:.3f}秒")

# 3. ファイルのメタ情報
print("\n=== 3. 件数確認 ===")
result = duckdb.sql("SELECT COUNT(*) AS total_rows FROM 'sales.csv'")
print(result)
```

実行結果:

```
=== 1. CSVを直接クエリ ===
┌───────┬─────────┬─────────┬──────────┬───────┐
│  id   │ product │ region  │ quantity │ price │
│ int64 │ varchar │ varchar │  int64   │ int64 │
├───────┼─────────┼─────────┼──────────┼───────┤
│     1 │ 商品B   │ 大阪    │       94 │   465 │
│     2 │ 商品C   │ 福岡    │       70 │  8204 │
│     3 │ 商品E   │ 福岡    │       78 │  6387 │
│     4 │ 商品B   │ 札幌    │       62 │  9332 │
│     5 │ 商品A   │ 東京    │        4 │  9868 │
└───────┴─────────┴─────────┴──────────┴───────┘

実行時間: 0.066秒

=== 2. 地域別・商品別の売上集計 ===
┌─────────┬─────────┬───────┬─────────────┐
│ region  │ product │ count │ total_sales │
│ varchar │ varchar │ int64 │   int128    │
├─────────┼─────────┼───────┼─────────────┤
│ 大阪    │ 商品A   │  4119 │  1056795590 │
│ 名古屋  │ 商品D   │  4077 │  1054438132 │
│ 名古屋  │ 商品E   │  4061 │  1046194128 │
│ 東京    │ 商品C   │  4101 │  1038086403 │
│ ...     │ ...     │   ... │         ... │
└─────────┴─────────┴───────┴─────────────┘

実行時間: 0.074秒
```

**10万件の集計が0.074秒**で完了しました。`FROM 'sales.csv'`と書くだけで、テーブル定義もインポート作業も不要です。

## Parquetファイルの操作

次にParquet形式を試します。ParquetはApacheが開発した列指向のバイナリフォーマットで、データ分析の世界では標準的なファイル形式です。DuckDBと同じ「列指向」なので相性が抜群です。

```python:04_parquet.py
import duckdb
import time
import os

# 1. CSVからParquetに変換
print("=== 1. CSV → Parquet 変換 ===")
duckdb.sql("COPY (SELECT * FROM 'sales.csv') TO 'sales.parquet' (FORMAT PARQUET)")
print("sales.parquet を作成しました")

# ファイルサイズ比較
csv_size = os.path.getsize('sales.csv') / 1024 / 1024
parquet_size = os.path.getsize('sales.parquet') / 1024 / 1024
print(f"CSV:     {csv_size:.2f} MB")
print(f"Parquet: {parquet_size:.2f} MB")
print(f"圧縮率:  {parquet_size/csv_size*100:.1f}%")

# 2. Parquetを直接クエリ
print("\n=== 2. Parquetを直接クエリ ===")
result = duckdb.sql("SELECT * FROM 'sales.parquet' LIMIT 5")
print(result)

# 3. パフォーマンス比較（集計クエリ）
print("\n=== 3. 集計クエリのパフォーマンス比較 ===")

query = """
    SELECT 
        region,
        product,
        SUM(quantity * price) AS total_sales
    FROM '{file}'
    GROUP BY region, product
    ORDER BY total_sales DESC
"""

# CSV
start = time.time()
for _ in range(10):
    duckdb.sql(query.format(file='sales.csv')).fetchall()
csv_time = time.time() - start

# Parquet
start = time.time()
for _ in range(10):
    duckdb.sql(query.format(file='sales.parquet')).fetchall()
parquet_time = time.time() - start

print(f"CSV:     {csv_time:.3f}秒（10回実行）")
print(f"Parquet: {parquet_time:.3f}秒（10回実行）")
print(f"Parquetは {csv_time/parquet_time:.1f}倍 高速")
```

実行結果:

```
=== 1. CSV → Parquet 変換 ===
sales.parquet を作成しました
CSV:     2.89 MB
Parquet: 0.76 MB
圧縮率:  26.3%

=== 3. 集計クエリのパフォーマンス比較 ===
CSV:     0.657秒（10回実行）
Parquet: 0.022秒（10回実行）
Parquetは 29.2倍 高速
```

結果をまとめると:

| 指標 | CSV | Parquet |
|------|-----|---------|
| ファイルサイズ | 2.89 MB | 0.76 MB |
| 集計クエリ（10回） | 0.657秒 | 0.022秒 |

Parquetは**ファイルサイズが約1/4**、**クエリ速度は約30倍高速**という結果になりました。列指向DB × 列指向ファイル形式の組み合わせの威力がよく分かります。

## 列指向の特性をもう少し詳しく

100万件のデータを使って、列指向の特性をさらに掘り下げてみます。

```python:05_columnar.py
import duckdb
import time

# 大きめのデータで検証（100万件）
print("=== 100万件のデータを生成 ===")
duckdb.sql("""
    COPY (
        SELECT 
            i AS id,
            'Product_' || (i % 100) AS product,
            'Region_' || (i % 10) AS region,
            (random() * 100)::INTEGER AS quantity,
            (random() * 10000)::INTEGER AS price,
            '2024-01-01'::DATE + ((i % 365)::INTEGER) AS sale_date,
            'Lorem ipsum dolor sit amet, consectetur adipiscing elit.' AS description
        FROM generate_series(1, 1000000) AS t(i)
    ) TO 'large_sales.parquet' (FORMAT PARQUET)
""")
print("large_sales.parquet を作成しました（100万件）")

# 1. 特定列のみの集計（列指向が得意なパターン）
print("\n=== 1. 特定列のみ集計（quantity, price） ===")
start = time.time()
result = duckdb.sql("""
    SELECT 
        SUM(quantity * price) AS total_sales,
        AVG(price) AS avg_price
    FROM 'large_sales.parquet'
""")
print(result)
print(f"実行時間: {time.time() - start:.3f}秒")
```

実行結果:

```
=== 100万件のデータを生成 ===
large_sales.parquet を作成しました（100万件）

=== 1. 特定列のみ集計（quantity, price） ===
┌──────────────┬─────────────┐
│ total_sales  │  avg_price  │
│    int128    │   double    │
├──────────────┼─────────────┤
│ 249789260048 │ 4998.477394 │
└──────────────┴─────────────┘

実行時間: 0.013秒
```

**100万件の集計が0.013秒**で完了しています。

列指向の特性を整理すると:

- 特定列の集計（SUM, AVG, COUNT等）が高速
- 列単位でデータが格納されているため、必要な列だけを読み込める
- 同じ型のデータが連続するため、圧縮効率が良い

一方、行指向（MySQL等）が得意な処理:

- 1レコード全体の取得（`SELECT * WHERE id = ?`）
- 1行単位の挿入・更新
- トランザクション処理

用途によって使い分けることが重要です。

## 永続化とコネクション

ここまではインメモリで動かしてきました。DuckDBはファイルにデータを永続化することもできます。

```python:06_persistence.py
import duckdb
import os

# 1. インメモリDB（デフォルト）
print("=== 1. インメモリDB ===")
con_memory = duckdb.connect()  # または duckdb.connect(':memory:')
con_memory.sql("CREATE TABLE test (id INTEGER, name VARCHAR)")
con_memory.sql("INSERT INTO test VALUES (1, 'memory')")
print(con_memory.sql("SELECT * FROM test"))
con_memory.close()
print("→ 接続を閉じるとデータは消える")

# 2. ファイルDB（永続化）
print("\n=== 2. ファイルDB（永続化） ===")
db_file = 'my_database.duckdb'

# 既存ファイルがあれば削除
if os.path.exists(db_file):
    os.remove(db_file)

# 作成と書き込み
con_file = duckdb.connect(db_file)
con_file.sql("CREATE TABLE users (id INTEGER, name VARCHAR, created_at TIMESTAMP)")
con_file.sql("INSERT INTO users VALUES (1, '田中', NOW())")
con_file.sql("INSERT INTO users VALUES (2, '鈴木', NOW())")
print(con_file.sql("SELECT * FROM users"))
con_file.close()

print(f"\nファイルサイズ: {os.path.getsize(db_file) / 1024:.1f} KB")

# 3. 再接続してデータ確認
print("\n=== 3. 再接続してデータ確認 ===")
con_file2 = duckdb.connect(db_file)
print(con_file2.sql("SELECT * FROM users"))
print(con_file2.sql("SHOW TABLES"))
con_file2.close()

# 4. 読み取り専用モード
print("\n=== 4. 読み取り専用モード ===")
con_readonly = duckdb.connect(db_file, read_only=True)
print(con_readonly.sql("SELECT * FROM users"))
try:
    con_readonly.sql("INSERT INTO users VALUES (3, '佐藤', NOW())")
except Exception as e:
    print(f"書き込みエラー: {type(e).__name__}")
con_readonly.close()
```

実行結果:

```
=== 1. インメモリDB ===
┌───────┬─────────┐
│  id   │  name   │
│ int32 │ varchar │
├───────┼─────────┤
│     1 │ memory  │
└───────┴─────────┘

→ 接続を閉じるとデータは消える

=== 2. ファイルDB（永続化） ===
┌───────┬─────────┬────────────────────────────┐
│  id   │  name   │         created_at         │
│ int32 │ varchar │         timestamp          │
├───────┼─────────┼────────────────────────────┤
│     1 │ 田中    │ 2026-02-01 08:55:06.642757 │
│     2 │ 鈴木    │ 2026-02-01 08:55:06.643862 │
└───────┴─────────┴────────────────────────────┘


ファイルサイズ: 524.0 KB

=== 3. 再接続してデータ確認 ===
（データが永続化されていることを確認）

=== 4. 読み取り専用モード ===
書き込みエラー: InvalidInputException
```

ファイルDBを使うメリットは永続化だけではありません。DuckDBはメモリに乗り切らない巨大なデータセットも、ディスクを活用して処理できます（これを「Out-of-Core処理」と呼びます）。これはPandasやSQLiteにはない大きな優位性です。搭載メモリ量を超えるデータの集計・分析が可能になります。

### ファイルロックの検証

DuckDBのファイルDBには重要な制約があります。あるプロセスが書き込み権限を持ってDBファイルを開いている間、他のプロセスは`read_only=True`であっても接続できません。この動作を検証してみましょう。

```python:09_file_lock_test.py
import duckdb
import os
import subprocess
import sys

DB_FILE = 'lock_test.duckdb'

def child_process_read():
    """子プロセス: read_only=True で接続を試みる"""
    try:
        con = duckdb.connect(DB_FILE, read_only=True)
        print("[子プロセス] read_only=True で接続成功")
        con.close()
    except Exception as e:
        print(f"[子プロセス] read_only=True で接続失敗: {type(e).__name__}")
        print(f"  エラー内容: {e}")

def child_process_write():
    """子プロセス: 書き込みモードで接続を試みる"""
    try:
        con = duckdb.connect(DB_FILE)
        print("[子プロセス] 書き込みモードで接続成功")
        con.close()
    except Exception as e:
        print(f"[子プロセス] 書き込みモードで接続失敗: {type(e).__name__}")
        print(f"  エラー内容: {e}")

if __name__ == '__main__':
    # 引数で子プロセスの動作を分岐
    if len(sys.argv) > 1:
        if sys.argv[1] == 'read':
            child_process_read()
        elif sys.argv[1] == 'write':
            child_process_write()
        sys.exit(0)
    
    # === 親プロセス ===
    print("=== ファイルロックの検証 ===\n")
    
    # 既存ファイルがあれば削除
    if os.path.exists(DB_FILE):
        os.remove(DB_FILE)
    
    # DBファイルを作成
    con = duckdb.connect(DB_FILE)
    con.sql("CREATE TABLE test (id INTEGER)")
    con.sql("INSERT INTO test VALUES (1)")
    print(f"[親プロセス] {DB_FILE} を書き込みモードで開いています")
    print("[親プロセス] 接続を保持したまま子プロセスを起動...\n")
    
    # 子プロセスから read_only=True で接続を試みる
    print("--- テスト1: 子プロセスから read_only=True で接続 ---")
    subprocess.run([sys.executable, __file__, 'read'])
    
    print()
    
    # 子プロセスから書き込みモードで接続を試みる
    print("--- テスト2: 子プロセスから書き込みモードで接続 ---")
    subprocess.run([sys.executable, __file__, 'write'])
    
    print()
    
    # 親プロセスの接続を閉じる
    con.close()
    print("[親プロセス] 接続を閉じました\n")
    
    # 接続を閉じた後に再度試行
    print("--- テスト3: 親プロセスが接続を閉じた後 ---")
    subprocess.run([sys.executable, __file__, 'read'])
    subprocess.run([sys.executable, __file__, 'write'])
    
    # クリーンアップ
    os.remove(DB_FILE)
```

実行結果:

```
=== ファイルロックの検証 ===

[親プロセス] lock_test.duckdb を書き込みモードで開いています
[親プロセス] 接続を保持したまま子プロセスを起動...

--- テスト1: 子プロセスから read_only=True で接続 ---
[子プロセス] read_only=True で接続失敗: IOException
  エラー内容: IO Error: Could not set lock on file "lock_test.duckdb": Conflicting lock is held in Python (PID 895)

--- テスト2: 子プロセスから書き込みモードで接続 ---
[子プロセス] 書き込みモードで接続失敗: IOException
  エラー内容: IO Error: Could not set lock on file "lock_test.duckdb": Conflicting lock is held in Python (PID 895)

[親プロセス] 接続を閉じました

--- テスト3: 親プロセスが接続を閉じた後 ---
[子プロセス] read_only=True で接続成功
[子プロセス] 書き込みモードで接続成功
```

| テスト | 状況 | 結果 |
|--------|------|------|
| テスト1 | 親が書き込みモードで接続中 → 子が`read_only=True`で接続 | **失敗** |
| テスト2 | 親が書き込みモードで接続中 → 子が書き込みモードで接続 | **失敗** |
| テスト3 | 親が接続を閉じた後 → 子が接続 | **成功** |

このように、DuckDBはファイルレベルでロックを取得するため、書き込みモードで接続中は他のプロセスからは一切アクセスできません。Webアプリケーションのように複数プロセスから同時にアクセスする用途には向いていないことが分かります。

接続方法をまとめると:

| メソッド | 説明 |
|----------|------|
| `duckdb.connect()` | インメモリDB（接続を閉じると消える） |
| `duckdb.connect(':memory:')` | 同上（明示的） |
| `duckdb.connect('file.duckdb')` | ファイル永続化 |
| `duckdb.connect('file.duckdb', read_only=True)` | 読み取り専用 |
| `duckdb.sql()` | グローバルなインメモリDBを使用 |

SQLiteと同じ感覚で使えるのが分かりますね。

:::message alert
**グローバル接続の注意点**

`duckdb.sql()` はグローバルなインメモリDBを使用するため、複数のスクリプトやスレッドから呼び出すと意図しない挙動になる可能性があります。ライブラリやパッケージ開発では `duckdb.connect()` で明示的に接続を作成することが推奨されています。
:::

## DuckDBならではの便利機能

MySQLにはない、DuckDBならではの機能をいくつか紹介します。

### 1. GLOB（複数ファイルの一括読み込み）

```python
# 複数CSVを一括で読み込み
result = duckdb.sql("""
    SELECT month, SUM(sales) AS total
    FROM 'sales_2025_*.csv'
    GROUP BY month
    ORDER BY month
""")
```

```
┌─────────┬────────┐
│  month  │ total  │
│ varchar │ int128 │
├─────────┼────────┤
│ 2025-01 │  46711 │
│ 2025-02 │  50342 │
│ 2025-03 │  52005 │
└─────────┴────────┘
```

`FROM 'sales_2025_*.csv'` のようにワイルドカードを使って、複数のファイルを一度にクエリできます。データレイク的な使い方に便利です。

### 2. EXCLUDE / REPLACE

```sql
-- 特定列を除外
SELECT * EXCLUDE (id) FROM products;

-- 列を変換して置換
SELECT * REPLACE (price * 1.1 AS price) FROM products;
```

```
EXCLUDE（idを除外）:
┌─────────┬───────┬──────────┐
│  name   │ price │ category │
│ varchar │ int32 │ varchar  │
├─────────┼───────┼──────────┤
│ Apple   │   100 │ A        │
│ Banana  │    80 │ B        │
│ Cherry  │   200 │ A        │
└─────────┴───────┴──────────┘


REPLACE（priceを1.1倍に）:
┌───────┬─────────┬───────────────┬──────────┐
│  id   │  name   │     price     │ category │
│ int32 │ varchar │ decimal(12,1) │ varchar  │
├───────┼─────────┼───────────────┼──────────┤
│     1 │ Apple   │         110.0 │ A        │
│     2 │ Banana  │          88.0 │ B        │
│     3 │ Cherry  │         220.0 │ A        │
└───────┴─────────┴───────────────┴──────────┘
```

列が多いテーブルを扱う際に、`SELECT`文を簡潔に書けます。

### 3. PIVOT / UNPIVOT

行と列を変換する機能がネイティブでサポートされています。

```sql
-- PIVOT（行→列変換）
PIVOT monthly_sales
ON month
USING SUM(sales)
GROUP BY region
```

```
┌─────────┬────────┬────────┬────────┐
│ region  │  1月   │  2月   │  3月   │
│ varchar │ int128 │ int128 │ int128 │
├─────────┼────────┼────────┼────────┤
│ Tokyo   │    100 │    120 │    130 │
│ Osaka   │     80 │     90 │    100 │
└─────────┴────────┴────────┴────────┘
```

```sql
-- UNPIVOT（列→行変換）
UNPIVOT wide_data
ON jan, feb, mar
INTO NAME month VALUE sales
```

```
┌─────────┬─────────┬───────┐
│ region  │  month  │ sales │
│ varchar │ varchar │ int32 │
├─────────┼─────────┼───────┤
│ Tokyo   │ jan     │   100 │
│ Tokyo   │ feb     │   120 │
│ Tokyo   │ mar     │   130 │
│ Osaka   │ jan     │    80 │
│ Osaka   │ feb     │    90 │
│ Osaka   │ mar     │   100 │
└─────────┴─────────┴───────┘
```

MySQLではCASE文を駆使する必要がある処理が、シンプルに書けます。

### 4. QUALIFY

ウィンドウ関数の結果で直接フィルタできます。

```sql
-- 各部門の売上トップを取得
SELECT dept, name, sales
FROM employee_sales
QUALIFY ROW_NUMBER() OVER (PARTITION BY dept ORDER BY sales DESC) = 1
```

```
┌───────────┬─────────┬───────┐
│   dept    │  name   │ sales │
│  varchar  │ varchar │ int32 │
├───────────┼─────────┼───────┤
│ Marketing │ Eve     │    90 │
│ Sales     │ Bob     │   150 │
└───────────┴─────────┴───────┘
```

通常、ウィンドウ関数の結果でフィルタするにはサブクエリが必要ですが、QUALIFYを使えば1クエリで完結します。

### 5. generate_series（連番生成）

`generate_series`は連続した値のシーケンスを生成するテーブル関数です。テストデータの作成に非常に便利です。

```sql
-- 1から10までの連番を生成
SELECT * FROM generate_series(1, 10);

-- ステップ指定（2刻み）
SELECT * FROM generate_series(1, 10, 2);
-- 結果: 1, 3, 5, 7, 9
```

本記事の検証でも、100万件のデータ生成に使用しました。

```sql
SELECT 
    i AS id,
    'Product_' || (i % 100) AS product,
    (random() * 100)::INTEGER AS quantity
FROM generate_series(1, 1000000) AS t(i)
```

`generate_series(1, 1000000)`で1〜100万の連番を生成し、`AS t(i)`でテーブル別名`t`、列名`i`を指定しています。`i`を使って各行のデータを計算できます。

MySQLには`generate_series`がないため、同じことをするには事前にシーケンステーブルを作成するか、再帰CTEを使うか、アプリケーション側でループしてINSERTする必要があります。DuckDBでは1行で大量のテストデータを生成できるので、検証や開発時に重宝します。

### 6. Pandas / Polars との連携

DuckDBはPythonのデータ分析ライブラリとの親和性が非常に高いです。

```python
import duckdb
import pandas as pd

# Pandas DataFrameを作成
df = pd.DataFrame({
    'name': ['田中', '鈴木', '佐藤'],
    'age': [28, 35, 22]
})

# Python変数をそのままSQLでクエリできる！
result = duckdb.sql("SELECT * FROM df WHERE age >= 25")
print(result)
```

```
┌─────────┬───────┐
│  name   │  age  │
│ varchar │ int64 │
├─────────┼───────┤
│ 田中    │    28 │
│ 鈴木    │    35 │
└─────────┴───────┘
```

Python上の変数`df`を、テーブル名のようにSQL内で直接参照できます（Replacement Scansという機能）。

また、クエリ結果を様々な形式に変換できます。

```python
result = duckdb.sql("SELECT * FROM df")

result.fetchall()   # Pythonリスト
result.df()         # Pandas DataFrame
result.pl()         # Polars DataFrame
result.arrow()      # Apache Arrow Table
```

PandasやPolarsで前処理したデータをDuckDBで集計し、結果をまたDataFrameに戻す、といったワークフローがシームレスに行えます。

## ユースケースまとめ

最後に、DuckDBがどういった場面で使えるかを整理します。以下の例は、本記事で作成した`sales.csv`を使って実際に試すことができます。

### 向いているケース

**1. 売上データの分析**

```python
# CSVを直接集計
result = duckdb.sql("""
    SELECT 
        region,
        COUNT(*) AS count,
        SUM(quantity * price) AS total_sales
    FROM 'sales.csv'
    GROUP BY region
    ORDER BY total_sales DESC
""")
```

テーブル定義もインポートも不要で、すぐに分析を始められます。

**2. ETL/データ変換パイプライン**

```python
# CSV読み込み → フィルタ・変換 → Parquet出力を1クエリで
duckdb.sql("""
    COPY (
        SELECT 
            region,
            product,
            quantity,
            price,
            quantity * price AS sales_amount
        FROM 'sales.csv'
        WHERE price >= 5000
    ) TO 'high_value_sales.parquet' (FORMAT PARQUET)
""")
```

複雑なデータ変換もSQLで記述でき、結果を様々なフォーマットで出力できます。

**3. 複数データソースの結合分析**

```python
# 商品マスタを動的に作成
duckdb.sql("""
    CREATE OR REPLACE TABLE products AS
    SELECT DISTINCT product, 
           CASE product 
               WHEN '商品A' THEN 'カテゴリ1'
               WHEN '商品B' THEN 'カテゴリ1'
               WHEN '商品C' THEN 'カテゴリ2'
               WHEN '商品D' THEN 'カテゴリ2'
               WHEN '商品E' THEN 'カテゴリ3'
           END AS category
    FROM 'sales.csv'
""")

# 売上データとJOINしてカテゴリ別集計
result = duckdb.sql("""
    SELECT 
        p.category,
        COUNT(*) AS count,
        SUM(s.quantity * s.price) AS total_sales
    FROM 'sales.csv' s
    JOIN products p ON s.product = p.product
    GROUP BY p.category
    ORDER BY total_sales DESC
""")
```

ファイルとテーブルをSQLでJOINできるのは非常に便利です。

### 向いていないケース

**1. 高頻度の単一行INSERT/UPDATE**

DuckDBはOLAP向けのため、1行ずつの書き込みは得意ではありません。WebアプリのバックエンドにはMySQL/PostgreSQLが適切です。

**2. 複数クライアントからの同時書き込み**

DuckDBはシングルライターモデルです。前述の「ファイルロックの検証」で確認した通り、あるプロセスが書き込み権限を持ってDBファイルを開いている間、他のプロセスは`read_only=True`であっても接続できません。

**3. トランザクション重視のアプリケーション**

銀行システムのような厳密なトランザクション管理が必要な場面には向いていません。

### 選択基準まとめ

| 用途 | DuckDB | MySQL/PostgreSQL |
|------|--------|------------------|
| データ分析 | ◎ | △ |
| ログ集計 | ◎ | ○ |
| ETL処理 | ◎ | △ |
| Webアプリ | × | ◎ |
| トランザクション | △ | ◎ |
| 同時書き込み | × | ◎ |
| セットアップ | ◎（即座） | △（要サーバー） |

## まとめ

DuckDBを実際に手を動かして検証した結果をまとめます。

### DuckDBの特徴

- **OLAP型データベース**: 大量データの集計・分析に特化
- **列指向**: 特定列の集計が高速、圧縮効率が良い
- **組み込み型**: `pip install`で即使える、サーバー不要
- **ファイル直接クエリ**: CSV/Parquetをインポートなしで分析可能

### 検証結果ハイライト

- 10万件のCSV集計: **0.074秒**
- 100万件のParquet集計: **0.013秒**
- CSV vs Parquet: Parquetは**約30倍高速**、サイズ**約1/4**

### 使いどころ

- ローカルでのデータ分析・探索
- ログファイルの集計
- ETL/データ変換パイプライン
- dbt/Dagsterなどとの連携

### 使わない場面

- Webアプリのバックエンド
- 高頻度のトランザクション処理
- 複数クライアントからの同時書き込み

---

今回は入門編として基本的な使い方を検証しました。

## 参考

- [DuckDB公式ドキュメント](https://duckdb.org/docs/)
- [DuckDB Python API](https://duckdb.org/docs/api/python/overview)
- [GitHub: duckdb/duckdb](https://github.com/duckdb/duckdb)
