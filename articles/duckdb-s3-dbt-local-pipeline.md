---
title: "DuckDB + S3 + dbt でローカルデータパイプラインを構築してみた"
emoji: "🦆"
type: "tech"
topics: ["duckdb", "dbt", "s3", "minio", "dataengineering"]
published: true
---

## はじめに

DuckDBは軽量・高速な分析特化型データベースとして注目を集めていますが、dbtと組み合わせたら**ローカル環境でどこまで実用的なデータパイプラインが作れるのか**が気になったので、実際に手を動かして検証してみました。

本記事では、S3互換ストレージ（MinIO）上のParquetファイルをDuckDBで直接クエリし、dbtで変換パイプラインを構築するまでの全手順を、つまづいたポイントも含めて共有します。AWSアカウント不要、Docker Composeだけで完結する構成です。

### リポジトリ

今回の検証で使用したコードはGitHubで公開しています。

https://github.com/toshiro3/duckdb-s3-dbt

### 対象読者

- DuckDBを分析以外にも活用したい方
- dbtの導入を検討しているデータエンジニア
- Athenaの代替としてローカル分析環境を探している方
- データパイプラインの学習をローカルで始めたい方

### 今回構築するパイプラインの全体像

```
S3（MinIO）上のParquetファイル
  ↓ DuckDB httpfs拡張で直接読み込み
dbt source定義（external_location）
  ↓
staging層（view）: 型変換・クレンジング
  ↓
marts層（table）: ビジネスロジック・集計
  ↓
DuckDBファイル（.duckdb）に永続化
```

### 使用する技術スタック

| ツール | バージョン | 役割 |
|--------|-----------|------|
| DuckDB | 1.4.4 | 分析エンジン |
| dbt-core | 1.11.2 | データ変換フレームワーク |
| dbt-duckdb | 1.10.0 | dbt ↔ DuckDB アダプター |
| MinIO | latest | S3互換オブジェクトストレージ |
| Python | 3.13 | 実行環境 |

## 1. 環境構築

### ディレクトリ構成

```
duckdb-s3-dbt/
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
└── workspace/
    ├── scripts/
    │   ├── seed_data.py
    │   └── test_duckdb_s3.py
    └── dbt_project/
        ├── dbt_project.yml
        ├── profiles.yml
        └── models/
            ├── staging/
            │   ├── _sources.yml
            │   ├── _schema.yml
            │   ├── stg_customers.sql
            │   ├── stg_products.sql
            │   └── stg_orders.sql
            └── marts/
                ├── _schema.yml
                ├── fct_order_details.sql
                ├── mart_monthly_sales.sql
                ├── mart_category_sales.sql
                └── mart_customer_summary.sql
```

### Docker Compose

MinIO（S3互換ストレージ）と作業用コンテナの2つを立てる構成にしました。

```yaml:docker-compose.yml
services:
  minio:
    image: minio/minio:latest
    container_name: minio
    ports:
      - "9000:9000"   # S3 API
      - "9001:9001"   # Web UI
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server /data --console-address ":9001"
    volumes:
      - minio-data:/data
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 5s
      timeout: 5s
      retries: 5

  workspace:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: duckdb-workspace
    volumes:
      - ./workspace:/workspace
    working_dir: /workspace
    ports:
      - "8080:8080"   # dbt docs serve
    depends_on:
      minio:
        condition: service_healthy
    environment:
      AWS_ACCESS_KEY_ID: minioadmin
      AWS_SECRET_ACCESS_KEY: minioadmin
      AWS_ENDPOINT_URL: http://minio:9000
      AWS_REGION: us-east-1
    tty: true
    stdin_open: true

volumes:
  minio-data:
```

```dockerfile:Dockerfile
FROM python:3.13-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    g++ \
    curl \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt /tmp/requirements.txt
RUN pip install --no-cache-dir -r /tmp/requirements.txt

WORKDIR /workspace

CMD ["tail", "-f", "/dev/null"]
```

```txt:requirements.txt
duckdb==1.4.4
dbt-duckdb==1.10.0
boto3==1.36.24
pyarrow==18.1.0
pandas==2.2.3
```

### 起動と動作確認

```bash
docker compose up -d --build
```

MinIO WebUIは http://localhost:9001 でアクセスできます（`minioadmin` / `minioadmin`）。

```bash
# バージョン確認
docker compose exec workspace bash -c "
  python -c 'import duckdb; print(f\"DuckDB: {duckdb.__version__}\")' && \
  dbt --version && \
  python -c 'import boto3; print(f\"boto3: {boto3.__version__}\")'
"
```

```
DuckDB: 1.4.4
Core:
  - installed: 1.11.2
  - latest:    1.11.2 - Up to date!

Plugins:
  - duckdb: 1.10.0 - Up to date!

boto3: 1.36.24
```

## 2. サンプルデータの投入

ECサイトを想定した3テーブルのデータを生成し、Parquet形式でMinIOにアップロードしていきます。

| テーブル | 件数 | 内容 |
|---------|------|------|
| customers | 100件 | 顧客マスタ（名前・都道府県・年齢・性別） |
| products | 20件 | 商品マスタ（商品名・カテゴリ・単価） |
| orders | 1,000件 | 注文トランザクション（顧客ID・商品ID・数量・ステータス） |

```python:workspace/scripts/seed_data.py
"""
MinIOにサンプルデータ（Parquet）を投入するスクリプト
"""

import random
from datetime import datetime, timedelta

import boto3
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq

ENDPOINT_URL = "http://minio:9000"
ACCESS_KEY = "minioadmin"
SECRET_KEY = "minioadmin"
BUCKET_NAME = "datalake"
REGION = "us-east-1"

random.seed(42)


def generate_customers(n: int = 100) -> pd.DataFrame:
    prefectures = [
        "東京都", "大阪府", "神奈川県", "愛知県", "福岡県",
        "北海道", "京都府", "兵庫県", "埼玉県", "千葉県",
    ]
    last_names = [
        "田中", "鈴木", "佐藤", "高橋", "伊藤",
        "渡辺", "山本", "中村", "小林", "加藤",
        "吉田", "山田", "松本", "井上", "木村",
    ]
    first_names = [
        "太郎", "花子", "一郎", "美咲", "健太",
        "さくら", "大輔", "陽子", "翔太", "真由美",
    ]

    customers = []
    for i in range(1, n + 1):
        registered_at = datetime(2023, 1, 1) + timedelta(days=random.randint(0, 730))
        customers.append({
            "customer_id": i,
            "customer_name": f"{random.choice(last_names)} {random.choice(first_names)}",
            "prefecture": random.choice(prefectures),
            "age": random.randint(18, 70),
            "gender": random.choice(["M", "F"]),
            "registered_at": registered_at.strftime("%Y-%m-%d"),
        })
    return pd.DataFrame(customers)


def generate_products(n: int = 20) -> pd.DataFrame:
    product_list = [
        ("ワイヤレスイヤホン", "家電", 4980),
        ("USBケーブル Type-C", "家電", 890),
        ("モバイルバッテリー", "家電", 3480),
        ("スマホケース", "アクセサリー", 1280),
        ("画面保護フィルム", "アクセサリー", 980),
        ("Bluetoothスピーカー", "家電", 6980),
        ("ノートPC スタンド", "PC周辺", 3980),
        ("マウスパッド", "PC周辺", 1480),
        ("Webカメラ", "PC周辺", 4980),
        ("メカニカルキーボード", "PC周辺", 12800),
        ("コーヒーメーカー", "キッチン", 8980),
        ("タンブラー 500ml", "キッチン", 2480),
        ("フライパン 26cm", "キッチン", 3280),
        ("トートバッグ", "ファッション", 2980),
        ("スニーカー", "ファッション", 7980),
        ("リュックサック", "ファッション", 5480),
        ("デスクライト LED", "インテリア", 4280),
        ("クッション", "インテリア", 1980),
        ("アロマディフューザー", "インテリア", 3680),
        ("ヨガマット", "スポーツ", 2480),
    ]

    products = []
    for i, (name, category, price) in enumerate(product_list[:n], 1):
        products.append({
            "product_id": i,
            "product_name": name,
            "category": category,
            "unit_price": price,
        })
    return pd.DataFrame(products)


def generate_orders(n: int = 1000, n_customers: int = 100, n_products: int = 20) -> pd.DataFrame:
    statuses = ["completed", "completed", "completed", "completed",
                "shipped", "shipped", "pending", "cancelled"]

    orders = []
    for i in range(1, n + 1):
        order_date = datetime(2024, 1, 1) + timedelta(days=random.randint(0, 364))
        orders.append({
            "order_id": i,
            "customer_id": random.randint(1, n_customers),
            "product_id": random.randint(1, n_products),
            "quantity": random.randint(1, 5),
            "status": random.choice(statuses),
            "order_date": order_date.strftime("%Y-%m-%d"),
        })
    return pd.DataFrame(orders)


def upload_parquet_to_minio(df: pd.DataFrame, key: str, s3_client) -> None:
    table = pa.Table.from_pandas(df)
    buf = pa.BufferOutputStream()
    pq.write_table(table, buf)

    s3_client.put_object(
        Bucket=BUCKET_NAME,
        Key=key,
        Body=buf.getvalue().to_pybytes(),
        ContentType="application/octet-stream",
    )
    print(f"  ✅ s3://{BUCKET_NAME}/{key} ({len(df)} rows)")


def main():
    s3 = boto3.client(
        "s3",
        endpoint_url=ENDPOINT_URL,
        aws_access_key_id=ACCESS_KEY,
        aws_secret_access_key=SECRET_KEY,
        region_name=REGION,
    )

    existing_buckets = [b["Name"] for b in s3.list_buckets()["Buckets"]]
    if BUCKET_NAME not in existing_buckets:
        s3.create_bucket(Bucket=BUCKET_NAME)
        print(f"🪣 バケット '{BUCKET_NAME}' を作成しました")

    print("\n📦 サンプルデータを生成・アップロード中...")
    customers_df = generate_customers()
    upload_parquet_to_minio(customers_df, "raw/customers.parquet", s3)
    products_df = generate_products()
    upload_parquet_to_minio(products_df, "raw/products.parquet", s3)
    orders_df = generate_orders()
    upload_parquet_to_minio(orders_df, "raw/orders.parquet", s3)

    print(f"\n🎉 完了！ customers: {len(customers_df)}, products: {len(products_df)}, orders: {len(orders_df)}")


if __name__ == "__main__":
    main()
```

```bash
docker compose exec workspace python scripts/seed_data.py
```

```
🪣 バケット 'datalake' を作成しました

📦 サンプルデータを生成・アップロード中...
  ✅ s3://datalake/raw/customers.parquet (100 rows)
  ✅ s3://datalake/raw/products.parquet (20 rows)
  ✅ s3://datalake/raw/orders.parquet (1000 rows)

🎉 完了！ customers: 100, products: 20, orders: 1000
```

MinIO WebUIでもデータが確認できます。

![MinIO WebUI](/images/duckdb-s3-dbt-local-pipeline/minio-webui.png)

## 3. DuckDBからS3上のデータを直接クエリ

dbtに入る前に、まずDuckDB単体でS3上のParquetファイルを直接クエリできるか試してみます。これがうまくいかないとdbt連携も始まらないので、先に確認しておきます。

### httpfs拡張の設定

DuckDBでS3上のファイルにアクセスするには、`httpfs`拡張のロードとS3接続情報の設定が必要です。MinIO向けにはいくつかハマりポイントがあったので、そのあたりも含めて記録しておきます。

```python
import duckdb

con = duckdb.connect()

con.execute("INSTALL httpfs;")
con.execute("LOAD httpfs;")

con.execute("""
    SET s3_region = 'us-east-1';
    SET s3_access_key_id = 'minioadmin';
    SET s3_secret_access_key = 'minioadmin';
    SET s3_endpoint = 'minio:9000';
    SET s3_use_ssl = false;
    SET s3_url_style = 'path';
""")
```

:::message
**MinIO接続のポイント**: `s3_url_style = 'path'` の設定が重要です。AWS S3のデフォルトは「仮想ホスト形式」（`bucket.s3.amazonaws.com`）ですが、MinIOでは「パス形式」（`s3.amazonaws.com/bucket`）を使う必要があります。この設定がないと、バケット名のDNS解決でエラーになります。
:::

:::message
**もう一つのハマりポイント**: `s3_endpoint` にはコンテナ名（`minio:9000`）を指定する必要があります。ホストマシンからは `localhost:9000` でアクセスできますが、workspaceコンテナからは `localhost` だと自分自身を指してしまいます。Docker Composeのサービス名がそのままコンテナ間のホスト名になる、というDocker Networkの基本ですが、S3設定の文脈では見落としがちです。
:::

### S3上のParquetを直接SELECT

設定が完了したので、S3パスを直接指定してクエリしてみます。

```sql
-- スキーマ確認
DESCRIBE SELECT * FROM 's3://datalake/raw/customers.parquet';

-- データプレビュー
SELECT * FROM 's3://datalake/raw/customers.parquet' LIMIT 5;
```

```
 customer_id customer_name prefecture  age gender registered_at
           1         鈴木 太郎        福岡県   33      M    2024-10-16
           2         山田 花子        埼玉県   23      F    2023-05-23
           3         田中 花子        愛知県   32      M    2023-02-02
           4         高橋 翔太        京都府   32      F    2024-07-28
           5         伊藤 太郎       神奈川県   62      F    2024-08-26
```

### 複数テーブルのJOIN

単一テーブルの読み込みはできたので、次はS3上の複数Parquetファイルを直接JOINできるか試してみます。

```sql
SELECT
    o.order_id,
    c.customer_name,
    p.product_name,
    p.category,
    o.quantity,
    o.quantity * p.unit_price AS total_amount,
    o.status,
    o.order_date
FROM 's3://datalake/raw/orders.parquet' AS o
JOIN 's3://datalake/raw/customers.parquet' AS c
    ON o.customer_id = c.customer_id
JOIN 's3://datalake/raw/products.parquet' AS p
    ON o.product_id = p.product_id
ORDER BY o.order_date DESC
LIMIT 5;
```

### 集計クエリ

JOINもいけたので、GROUP BY + 集計関数も試します。

```sql
SELECT
    p.category,
    COUNT(*) AS order_count,
    SUM(o.quantity * p.unit_price) AS total_revenue,
    ROUND(AVG(o.quantity * p.unit_price), 0) AS avg_order_amount
FROM 's3://datalake/raw/orders.parquet' AS o
JOIN 's3://datalake/raw/products.parquet' AS p
    ON o.product_id = p.product_id
WHERE o.status != 'cancelled'
GROUP BY p.category
ORDER BY total_revenue DESC;
```

```
category  order_count  total_revenue  avg_order_amount
    PC周辺          173      3086420.0           17841.0
  ファッション          131      2085180.0           15917.0
      家電          156      1958510.0           12555.0
    キッチン          127      1915600.0           15083.0
   インテリア          122      1178560.0            9660.0
    スポーツ           45       386880.0            8597.0
  アクセサリー          104       321400.0            3090.0
```

DuckDB単体でS3上のParquetに対して、スキーマ確認・フィルタ・JOIN・集計のすべてが問題なく動きました。ローカルにデータをダウンロードする必要がなく、S3パスを直接指定するだけでクエリできるのは想像以上に快適です。

## 4. dbt-duckdbプロジェクトの構築

DuckDB単体でのS3クエリが確認できたので、ここからdbtを組み合わせていきます。

ここで押さえておきたいのが、**dbt自体はS3にアクセスしない**という点です。実際にS3上のParquetを読み込んでいるのはDuckDBの`httpfs`拡張であり、dbtの役割はSQLの生成・実行管理・テスト・ドキュメント生成です。ステップ3でPythonスクリプトから手動で行っていたS3設定を、dbt-duckdbアダプターが接続時に自動で行ってくれる、という関係になっています。

```
dbt（SQL生成・実行管理）
  ↓ SQLを渡す
DuckDB（クエリエンジン）
  ↓ httpfs拡張でデータ取得
S3（MinIO）上のParquetファイル
```

### プロジェクト設定

```yaml:workspace/dbt_project/dbt_project.yml
name: 'duckdb_s3_pipeline'
version: '1.0.0'

profile: 'duckdb_s3_pipeline'

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

clean-targets:
  - "target"
  - "dbt_packages"
```

### profiles.yml — DuckDB + S3接続設定

ここが今回の検証で一番重要な設定ファイルです。`extensions`でhttpfsとparquetを、`settings`でMinIO向けのS3接続情報を指定します。

```yaml:workspace/dbt_project/profiles.yml
duckdb_s3_pipeline:
  target: dev
  outputs:
    dev:
      type: duckdb
      path: /workspace/dbt_project/duckdb_s3.duckdb
      threads: 4
      extensions:
        - httpfs
        - parquet
      settings:
        s3_region: "us-east-1"
        s3_access_key_id: "minioadmin"
        s3_secret_access_key: "minioadmin"
        s3_endpoint: "minio:9000"
        s3_use_ssl: false
        s3_url_style: "path"
```

:::message
**ポイント**: ステップ3のPythonスクリプトで手動設定していた `INSTALL httpfs; LOAD httpfs; SET s3_endpoint = ...` といった一連の処理が、profiles.ymlの`extensions`と`settings`だけで完結します。dbt-duckdbアダプターがDuckDB接続時に自動的に適用してくれるので、dbtモデル内でSET文を書く必要はありません。
:::

:::message
**dev/prod切り替えのTips**: dbt-duckdbには `external_root` という設定があり、source定義の `external_location` のベースパスを環境ごとに切り替えられます。たとえば、devではMinIO（`s3://datalake`）、prodでは本物のS3（`s3://my-prod-bucket`）という使い分けが、profiles.ymlのtarget切り替えだけで実現できます。
```yaml
# profiles.yml
duckdb_s3_pipeline:
  outputs:
    dev:
      type: duckdb
      path: /workspace/dbt_project/duckdb_s3.duckdb
      external_root: "s3://datalake"  # MinIO
      # ...
    prod:
      type: duckdb
      path: /workspace/dbt_project/duckdb_s3.duckdb
      external_root: "s3://my-prod-bucket"  # 本番S3
      # ...
```
:::

### source定義 — S3上のParquetをdbtのsourceとして認識

dbt-duckdbでは、`meta.external_location`を使ってS3上のファイルをsourceとして定義できます。この仕組みが今回の構成のキーになる部分なので、実際に動くか確認していきます。

```yaml:workspace/dbt_project/models/staging/_sources.yml
version: 2

sources:
  - name: raw
    description: "MinIO（S3互換）上のrawデータ"
    meta:
      external_location: "s3://datalake/raw/{name}.parquet"
    tables:
      - name: customers
        description: "顧客マスタ"
        columns:
          - name: customer_id
            description: "顧客ID"
            tests:
              - unique
              - not_null
          - name: customer_name
            description: "顧客名"
          - name: prefecture
            description: "都道府県"
          - name: age
            description: "年齢"
          - name: gender
            description: "性別（M/F）"
          - name: registered_at
            description: "登録日"

      - name: products
        description: "商品マスタ"
        columns:
          - name: product_id
            description: "商品ID"
            tests:
              - unique
              - not_null
          - name: product_name
            description: "商品名"
          - name: category
            description: "商品カテゴリ"
          - name: unit_price
            description: "単価（円）"

      - name: orders
        description: "注文トランザクション"
        columns:
          - name: order_id
            description: "注文ID"
            tests:
              - unique
              - not_null
          - name: customer_id
            description: "顧客ID（FK）"
            tests:
              - not_null
          - name: product_id
            description: "商品ID（FK）"
            tests:
              - not_null
          - name: status
            description: "注文ステータス"
            tests:
              - accepted_values:
                  arguments:
                    values: ["completed", "shipped", "pending", "cancelled"]
          - name: order_date
            description: "注文日"
```

:::message
**`external_location`のパターン**: `s3://datalake/raw/{name}.parquet`の`{name}`は、`tables`配下の各テーブル名に自動的に置換されます。つまり`customers`テーブルは`s3://datalake/raw/customers.parquet`として解決されます。テーブルごとに個別のパスを指定することも可能です。

具体的には、dbtモデル内の `{{ source('raw', 'customers') }}` がSQL生成時に `'s3://datalake/raw/customers.parquet'` というS3パスに展開され、それをDuckDBがhttpfs経由で読み込む、という流れです。
:::

### 接続テスト

```bash
docker compose exec workspace bash -c "cd /workspace/dbt_project && dbt debug"
```

```
Configuration:
  profiles.yml file [OK found and valid]
  dbt_project.yml file [OK found and valid]

Connection:
  database: duckdb_s3
  schema: main
  path: /workspace/dbt_project/duckdb_s3.duckdb
  extensions: ['httpfs', 'parquet']
  settings: {'s3_region': 'us-east-1', ...}
  Connection test: [OK connection ok]
```

## 5. dbtモデルの実装

### モデルのDAG（依存関係）

`dbt docs serve` で生成されるLineage Graphで、モデル間の依存関係を可視化できます。

![dbt Lineage Graph](/images/duckdb-s3-dbt-local-pipeline/dbt-lineage-graph.png)

source（緑）→ staging（水色）→ fct（ピンク）→ marts（水色）という流れが一目でわかります。

### staging層 — 型変換とクレンジング

staging層はviewとして作成し、S3のParquetから読み込んだデータの型変換を行います。Parquetから読み込んだ日付カラムがVARCHAR型になっていたので、ここでDATE型にキャストしておきます。

```sql:models/staging/stg_customers.sql
{{
    config(
        materialized='view'
    )
}}

SELECT
    customer_id,
    customer_name,
    prefecture,
    age,
    gender,
    CAST(registered_at AS DATE) AS registered_at
FROM {{ source('raw', 'customers') }}
```

```sql:models/staging/stg_products.sql
{{
    config(
        materialized='view'
    )
}}

SELECT
    product_id,
    product_name,
    category,
    unit_price
FROM {{ source('raw', 'products') }}
```

```sql:models/staging/stg_orders.sql
{{
    config(
        materialized='view'
    )
}}

SELECT
    order_id,
    customer_id,
    product_id,
    quantity,
    status,
    CAST(order_date AS DATE) AS order_date
FROM {{ source('raw', 'orders') }}
```

staging層のスキーマ定義では、リレーションシップテスト（外部キー整合性チェック）も設定してみました。dbtでデータ品質テストまで一気通貫で回せるか確認するためです。

```yaml:models/staging/_schema.yml
version: 2

models:
  - name: stg_customers
    description: "顧客マスタ（staging）- 型変換済み"
    columns:
      - name: customer_id
        tests:
          - unique
          - not_null

  - name: stg_products
    description: "商品マスタ（staging）- 型変換済み"
    columns:
      - name: product_id
        tests:
          - unique
          - not_null

  - name: stg_orders
    description: "注文トランザクション（staging）- 型変換済み"
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: customer_id
        tests:
          - not_null
          - relationships:
              arguments:
                to: ref('stg_customers')
                field: customer_id
      - name: product_id
        tests:
          - not_null
          - relationships:
              arguments:
                to: ref('stg_products')
                field: product_id
```

:::message alert
**dbt 1.11でハマったポイント**: `accepted_values`や`relationships`テストで、引数を`arguments`プロパティの下にネストしないとdeprecation warningが出ます。ネット上の記事やチュートリアルは旧記法のものが多いので注意が必要です。
```yaml
# ❌ 旧: トップレベルに直接記述
- relationships:
    to: ref('stg_customers')
    field: customer_id

# ✅ 新: argumentsの下にネスト
- relationships:
    arguments:
      to: ref('stg_customers')
      field: customer_id
```
:::

### marts層 — ビジネスロジックと集計

marts層はtableとして作成し、DuckDBファイルに永続化します。staging層のviewを参照しているので、実行時にS3からのデータ取得→変換→永続化が一連の流れで行われるはずです。

:::message
**materialized戦略の補足**: 今回は staging=view、marts=table としましたが、DuckDBはインメモリ処理が非常に速いため、中間モデルを `materialized='ephemeral'`（CTEとして展開、テーブルもviewも作らない）にする選択肢もあります。ディスクI/O（.duckdbファイルへの書き込み）を完全にスキップできるため、使い捨てのアドホック分析ではさらに高速化できます。ただし、ephemeralモデルは `dbt docs` のLineage Graphに表示されず、デバッグもしにくいので、今回のように再利用する中間テーブルにはtableやviewの方が適しています。
:::

#### fct_order_details — 注文明細ファクトテーブル

まず、顧客・商品情報を結合した非正規化テーブルを作ります。これをベースに、後続の集計モデルを組み立てていきます。

```sql:models/marts/fct_order_details.sql
{{
    config(
        materialized='table'
    )
}}

SELECT
    o.order_id,
    o.order_date,
    o.status,
    o.customer_id,
    c.customer_name,
    c.prefecture,
    c.age,
    c.gender,
    o.product_id,
    p.product_name,
    p.category,
    p.unit_price,
    o.quantity,
    o.quantity * p.unit_price AS total_amount
FROM {{ ref('stg_orders') }} AS o
INNER JOIN {{ ref('stg_customers') }} AS c
    ON o.customer_id = c.customer_id
INNER JOIN {{ ref('stg_products') }} AS p
    ON o.product_id = p.product_id
```

#### mart_monthly_sales — 月別売上サマリ

```sql:models/marts/mart_monthly_sales.sql
{{
    config(
        materialized='table'
    )
}}

SELECT
    DATE_TRUNC('month', order_date) AS order_month,
    COUNT(*) AS order_count,
    COUNT(DISTINCT customer_id) AS unique_customers,
    SUM(quantity) AS total_quantity,
    SUM(total_amount) AS total_revenue,
    ROUND(AVG(total_amount), 0) AS avg_order_amount
FROM {{ ref('fct_order_details') }}
WHERE status != 'cancelled'
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY order_month
```

#### mart_category_sales — カテゴリ別売上サマリ

```sql:models/marts/mart_category_sales.sql
{{
    config(
        materialized='table'
    )
}}

SELECT
    category,
    COUNT(*) AS order_count,
    COUNT(DISTINCT customer_id) AS unique_customers,
    SUM(quantity) AS total_quantity,
    SUM(total_amount) AS total_revenue,
    ROUND(AVG(total_amount), 0) AS avg_order_amount,
    ROUND(
        SUM(total_amount) * 100.0 / SUM(SUM(total_amount)) OVER (),
        1
    ) AS revenue_share_pct
FROM {{ ref('fct_order_details') }}
WHERE status != 'cancelled'
GROUP BY category
ORDER BY total_revenue DESC
```

#### mart_customer_summary — 顧客別購買サマリ

```sql:models/marts/mart_customer_summary.sql
{{
    config(
        materialized='table'
    )
}}

SELECT
    customer_id,
    customer_name,
    prefecture,
    age,
    gender,
    COUNT(*) AS order_count,
    SUM(total_amount) AS lifetime_revenue,
    ROUND(AVG(total_amount), 0) AS avg_order_amount,
    MIN(order_date) AS first_order_date,
    MAX(order_date) AS last_order_date,
    DATEDIFF('day', MIN(order_date), MAX(order_date)) AS active_days
FROM {{ ref('fct_order_details') }}
WHERE status != 'cancelled'
GROUP BY customer_id, customer_name, prefecture, age, gender
ORDER BY lifetime_revenue DESC
```

## 6. パイプラインの実行と検証

### dbt run

```bash
docker compose exec workspace bash -c "cd /workspace/dbt_project && dbt run"
```

```
Concurrency: 4 threads (target='dev')

1 of 7 START sql view model main.stg_customers ............... [RUN]
2 of 7 START sql view model main.stg_orders .................. [RUN]
3 of 7 START sql view model main.stg_products ................ [RUN]
1 of 7 OK created sql view model main.stg_customers .......... [OK in 0.08s]
2 of 7 OK created sql view model main.stg_orders ............. [OK in 0.08s]
3 of 7 OK created sql view model main.stg_products ........... [OK in 0.08s]
4 of 7 START sql table model main.fct_order_details .......... [RUN]
4 of 7 OK created sql table model main.fct_order_details ..... [OK in 0.05s]
5 of 7 START sql table model main.mart_category_sales ........ [RUN]
6 of 7 START sql table model main.mart_customer_summary ...... [RUN]
7 of 7 START sql table model main.mart_monthly_sales ......... [RUN]
5 of 7 OK created sql table model main.mart_category_sales ... [OK in 0.06s]
6 of 7 OK created sql table model main.mart_customer_summary . [OK in 0.06s]
7 of 7 OK created sql table model main.mart_monthly_sales .... [OK in 0.05s]

Done. PASS=7 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=7
```

7モデルすべてが**0.31秒**で完了しました。staging（view）→ marts（table）の依存関係に従って正しい順序で実行されていて、スレッド数4で依存関係のないモデルが並列実行されているのもログから読み取れます。

### dbt test

```bash
docker compose exec workspace bash -c "cd /workspace/dbt_project && dbt test"
```

```
Done. PASS=28 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=28
```

28テストすべてPASS。sourceレベルのデータ品質テスト（unique, not_null, accepted_values）からstagingレベルのリレーションシップテストまで、一発で全部通りました。

### dbt docs generate

```bash
docker compose exec workspace bash -c "cd /workspace/dbt_project && dbt docs generate"
```

```
Catalog written to /workspace/dbt_project/target/catalog.json
```

`dbt docs serve --port 8080 --host 0.0.0.0` でドキュメントサーバーを起動すると、ブラウザで各モデルの詳細やカラム定義、テスト情報が確認できます。

![dbt docs](/images/duckdb-s3-dbt-local-pipeline/dbt-docs.png)

### 実行結果の確認

DuckDBファイルに永続化されたmartsテーブルの中身を直接クエリして、意図通りの集計結果が入っているか確認してみます。

```python
import duckdb
con = duckdb.connect('/workspace/dbt_project/duckdb_s3.duckdb', read_only=True)

print(con.execute('SELECT * FROM main.mart_monthly_sales').fetchdf())
print(con.execute('SELECT * FROM main.mart_category_sales').fetchdf())

con.close()
```

**月別売上サマリ（mart_monthly_sales）**

| order_month | order_count | unique_customers | total_revenue | avg_order_amount |
|-------------|-------------|------------------|---------------|------------------|
| 2024-01 | 60 | 49 | 683,840 | 11,397 |
| 2024-02 | 71 | 53 | 912,910 | 12,858 |
| 2024-03 | 80 | 55 | 1,033,880 | 12,924 |
| ... | ... | ... | ... | ... |
| 2024-12 | 62 | 47 | 889,340 | 14,344 |

**カテゴリ別売上サマリ（mart_category_sales）**

| category | order_count | total_revenue | revenue_share_pct |
|----------|-------------|---------------|-------------------|
| PC周辺 | 173 | 3,086,420 | 28.2% |
| ファッション | 131 | 2,085,180 | 19.1% |
| 家電 | 156 | 1,958,510 | 17.9% |
| キッチン | 127 | 1,915,600 | 17.5% |
| インテリア | 122 | 1,178,560 | 10.8% |
| スポーツ | 45 | 386,880 | 3.5% |
| アクセサリー | 104 | 321,400 | 2.9% |

## 7. Athena代替としてのユースケース比較

今回の検証を通じて、DuckDB + dbtの構成がAWS Athenaの代替としてどの程度使えるのか、実感をもとに整理してみます。

### DuckDB + dbt vs Athena の比較

| 観点 | DuckDB + dbt（ローカル） | Athena |
|------|--------------------------|--------|
| コスト | **無料** | スキャンデータ量に応じた従量課金 |
| セットアップ | Docker Composeのみ | AWSアカウント、IAM、Glue Catalog等 |
| 実行速度 | 小〜中規模データは**非常に高速** | コールドスタートあり |
| スケーラビリティ | マシンのメモリに依存 | 大規模データでもスケール |
| S3連携 | httpfs拡張で直接アクセス | ネイティブ対応 |
| dbt連携 | dbt-duckdbアダプター | dbt-athenaアダプター |
| 適したデータ量 | 〜数十GB | 数十GB〜PB級 |

### DuckDBが向いていると感じたケース

- **ローカル開発・プロトタイピング**: S3上のデータを手元で素早く探索・検証したいとき
- **CI/CDでのデータテスト**: dbt testをローカルやGitHub Actionsで回したいとき
- **小〜中規模のバッチ処理**: 定期的なデータ変換・集計（〜数十GB程度）
- **Athenaの開発環境代替**: 本番はAthena、開発はDuckDBという使い分け
- **コスト削減**: 開発中のクエリ実行でAthenaの課金を避けたい場合

### Athenaの方が向いているケース

- **大規模データ（数百GB〜PB）**: DuckDBのメモリ制約を超えるデータ
- **マルチユーザー環境**: チームでの共有クエリ実行
- **AWSサービスとの統合**: Glue, Step Functions, QuickSight等との連携
- **サーバーレス要件**: インフラ管理不要な環境が必要な場合

## まとめ

DuckDB + S3（MinIO）+ dbt でローカル完結のデータパイプラインを構築し、実用性を検証しました。

### 検証結果のまとめ

| 項目 | 結果 |
|------|------|
| DuckDBからS3（MinIO）への直接クエリ | ✅ httpfs拡張でParquet/CSVを直接SELECT可能 |
| S3上の複数ファイルのJOIN | ✅ 問題なく動作 |
| dbt-duckdbアダプターの接続 | ✅ profiles.ymlのextensions/settingsで設定完結 |
| external_locationによるsource定義 | ✅ `{name}`パターンで複数テーブル一括定義可能 |
| staging → marts のパイプライン実行 | ✅ 7モデル・28テスト全PASS（0.31秒） |
| DuckDBファイルへの永続化 | ✅ martsテーブルがDuckDBファイルに保存 |

### 検証を通じてわかったポイント

1. **MinIO向けの`s3_url_style: path`設定**がないとバケットのDNS解決で詰まる
2. **`s3_endpoint`にはコンテナ名を指定**（`localhost`ではなく`minio`）— Docker Networkの基本だが見落としがち
3. **`meta.external_location`の`{name}`パターン**で複数ソースを簡潔に定義できて便利
4. **staging層はview、marts層はtable**で使い分けるとS3アクセスが最小限で済む（ephemeralも選択肢）
5. **`external_root`を使えばdev/prod切り替え**がprofiles.ymlのtarget変更だけで対応できる
6. **dbt 1.11以降**は`arguments`プロパティへのネストが必要（既存記事の記法だとwarningが出る）
