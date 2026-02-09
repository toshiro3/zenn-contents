---
title: "BigQuery + dbt でデータパイプラインを構築してみた【DuckDB + dbt との比較付き】"
emoji: "🔄"
type: "tech"
topics: ["bigquery", "dbt", "dataengineering", "gcp", "sql"]
published: false
---

## はじめに

以前「[DuckDB + S3 + dbt でローカルデータパイプラインを構築してみた](https://zenn.dev/toshiro3/articles/duckdb-s3-dbt-local-pipeline)」という記事で、DuckDB をターゲットにした dbt パイプラインを構築しました。dbt の基本的な使い方は分かったので、今回は**BigQuery をターゲットにしたらどうなるのか**を試してみます。

「[【BigQuery入門】DuckDBユーザーが初めてクラウドDWHを触ってみた](https://zenn.dev/toshiro3/articles/bigquery-introduction)」で作成済みのデータセット・テーブルがあるので、それをそのまま活用します。DuckDB + dbt との違いを意識しながら進めていきます。

### やったこと

- dbt-bigquery のセットアップとサービスアカウント認証
- 既存の BigQuery テーブルをソースにした staging → mart の2層構成
- BigQuery 固有のマテリアライゼーション（table / view / incremental）
- パーティショニング・クラスタリングの dbt 設定
- テスト（not_null / unique / relationships）

### 全体のデータフロー

今回構築するパイプラインの全体像です。

```
[BigQuery Tables]                [dbt seed]
 users / orders / access_logs    daily_access_summary
         |                              |
         v                              |
   +---------------+                    |
   | Staging (view) |                   |
   |  stg_users     |                   |
   |  stg_orders    |                   |
   +-------+-------+                    |
           |                            |
           v                            v
   +--------------------------------------------+
   |            Mart (table)                     |
   |  mart_user_orders                           |
   |  mart_daily_access        <- partitioned    |
   |  mart_daily_access_incr.  <- incremental    |
   +--------------------------------------------+
```

### 検証環境

| 項目 | バージョン / 値 |
|---|---|
| dbt-core | 1.11.3 |
| dbt-bigquery | 1.11.0 |
| Python | 3.12.12（uv で管理） |
| GCP プロジェクト | my-bq-project |
| データセット | bq_tutorial（asia-northeast1） |
| 既存テーブル | users, orders, access_logs |

### リポジトリ

https://github.com/toshiro3/bigquery-dbt

## Step 1: 環境構築

### プロジェクトのセットアップ

まずは uv でプロジェクトを作成して dbt-bigquery をインストールします。

```bash
mkdir bigquery-dbt && cd bigquery-dbt
uv init --python 3.12
uv python install 3.12
uv add dbt-core dbt-bigquery
```

:::message alert
執筆時点（2026年2月）では、dbt の依存ライブラリ（mashumaro）が Python 3.14 に未対応でした。3.14 で `dbt init` を実行すると `UnserializableField` エラーが出るため、**Python 3.12 を指定**しています。
:::

dbt プロジェクトを初期化します。

```bash
uv run dbt init bigquery_dbt
```

対話プロンプトで `[1] bigquery` → `[2] service_account` を選択しました。

### サービスアカウントの準備

認証にはサービスアカウントキーを使いました。GCP コンソールで以下の手順で作成します。

1. **IAMと管理** → **サービスアカウント** → **作成**
2. ロールに **BigQuery データ編集者** と **BigQuery ジョブユーザー** を付与
3. **鍵を追加** → **JSON** でキーファイルをダウンロード
4. `~/.config/gcloud/dbt-service-account.json` に配置

:::message alert
キーファイルは絶対に Git にコミットしないように注意。プロジェクト外に配置するのが安全です。
:::

DuckDB + dbt ではローカルファイルを指定するだけで認証不要だったので、ここが最初の大きな違いです。

### profiles.yml の設定

`dbt init` で `~/.dbt/profiles.yml` が生成されますが、**location の選択肢に東京リージョンがありません**でした。`[1] US` か `[2] EU` しか選べないので、とりあえず US を選んで生成後に手動で `asia-northeast1` に修正しました。

```yaml:~/.dbt/profiles.yml
bigquery_dbt:
  outputs:
    dev:
      type: bigquery
      method: service-account
      project: my-bq-project
      dataset: bq_tutorial
      location: asia-northeast1  # ← 手動で修正
      keyfile: /path/to/your-service-account-key.json
      job_execution_timeout_seconds: 300
      job_retries: 1
      priority: interactive
      threads: 1
  target: dev
```

`location` はデータセットのリージョンと一致させる必要があります。不一致だとクエリ実行時にエラーになるので注意です。

:::message
`threads: 1` は今回の検証では十分ですが、BigQuery は並列実行に強いので、モデル数が増えてきたら `4` 程度に上げるとスループットが改善します。
:::

### 接続確認

```bash
cd bigquery_dbt
uv run dbt debug
```

```
Connection:
  method: service-account
  database: my-bq-project
  schema: bq_tutorial
  location: asia-northeast1
  ...
  Connection test: [OK connection ok]

All checks passed!
```

通りました。DuckDB の `dbt debug` はファイルの存在チェック程度でしたが、BigQuery では **GCP API への実接続テスト**が走ります。サービスアカウントの権限不足があればここで気づけます。

ここまでの DuckDB + dbt との違いをまとめると：

| | DuckDB + dbt | BigQuery + dbt |
|---|---|---|
| 認証 | 不要（ローカルファイル） | サービスアカウントキー or OAuth |
| 接続先 | ローカルの .duckdb ファイル | GCP プロジェクト + データセット |
| location 指定 | 不要 | 必須（リージョン一致） |
| `dbt debug` | ファイル存在チェック | GCP API 実接続テスト |

## Step 2: ソース定義

BigQuery 入門で作成した既存テーブルを dbt のソースとして定義します。

```yaml:models/staging/_sources.yml
version: 2

sources:
  - name: bq_tutorial
    database: my-bq-project
    schema: bq_tutorial
    tables:
      - name: users
      - name: orders
      - name: access_logs
```

```bash
uv run dbt list --resource-type source
```

```
source:bigquery_dbt.bq_tutorial.access_logs
source:bigquery_dbt.bq_tutorial.orders
source:bigquery_dbt.bq_tutorial.users
```

3つ認識されました。ここは DuckDB + dbt と同じ書き方です。

## Step 3: Staging モデルの作成

staging モデルを作る前に、テーブルのカラム構成を確認しておきます。`dbt show --inline` で INFORMATION_SCHEMA を引くと便利でした。

```bash
uv run dbt show --inline "select column_name from \
  \`my-bq-project\`.bq_tutorial.INFORMATION_SCHEMA.COLUMNS \
  where table_name = 'users'"
```

```
| column_name |
| ----------- |
| user_id     |
| name        |
| email       |
| age         |
| prefecture  |
```

```bash
uv run dbt show --inline "select column_name from \
  \`my-bq-project\`.bq_tutorial.INFORMATION_SCHEMA.COLUMNS \
  where table_name = 'orders'"
```

```
| column_name  |
| ------------ |
| order_id     |
| user_id      |
| product_name |
| price        |
| quantity     |
```

カラム構成が分かったので、staging モデルを作成します。

```sql:models/staging/stg_users.sql
with source as (
    select * from {{ source('bq_tutorial', 'users') }}
)

select
    user_id,
    name as user_name,
    email,
    age,
    prefecture
from source
```

```sql:models/staging/stg_orders.sql
with source as (
    select * from {{ source('bq_tutorial', 'orders') }}
)

select
    order_id,
    user_id,
    product_name,
    price,
    quantity,
    price * quantity as total_amount
from source
```

`stg_orders` では `price * quantity` の計算列を追加しました。

```bash
uv run dbt run
```

```
1 of 2 OK created sql view model bq_tutorial.stg_orders ... [CREATE VIEW (0 processed) in 1.06s]
2 of 2 OK created sql view model bq_tutorial.stg_users .... [CREATE VIEW (0 processed) in 1.09s]

Done. PASS=2 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=2
```

デフォルトのマテリアライゼーションは `view` なので、ビューとして作成されます。DuckDB + dbt と同じですが、ログに **BigQuery のジョブ URL** が表示されるのが違いますね。クリックすると GCP コンソールで詳細を確認できます。

## Step 4: Mart モデルの作成

staging モデルを JOIN して、ユーザーごとの注文サマリーを作ります。

```sql:models/mart/mart_user_orders.sql
with users as (
    select * from {{ ref('stg_users') }}
),

orders as (
    select * from {{ ref('stg_orders') }}
),

user_orders as (
    select
        u.user_id,
        u.user_name,
        u.prefecture,
        count(o.order_id) as order_count,
        sum(o.total_amount) as total_spent,
        avg(o.total_amount) as avg_order_amount
    from users u
    left join orders o on u.user_id = o.user_id
    group by u.user_id, u.user_name, u.prefecture
)

select * from user_orders
```

mart 層はテーブルとして実体化させたいので、`dbt_project.yml` で設定しました。

```yaml:dbt_project.yml
models:
  bigquery_dbt:
    staging:
      +materialized: view
    mart:
      +materialized: table
```

```bash
uv run dbt run
```

```
1 of 3 OK created sql view model bq_tutorial.stg_orders ........... [CREATE VIEW (0 processed) in 1.01s]
2 of 3 OK created sql view model bq_tutorial.stg_users ............ [CREATE VIEW (0 processed) in 0.93s]
3 of 3 OK created sql table model bq_tutorial.mart_user_orders .... [CREATE TABLE (5.0 rows, 485.0 Bytes processed) in 2.82s]

Done. PASS=3 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=3
```

staging は `view`、mart は `table` で作成されました。ログに `485.0 Bytes processed` とスキャン量が出ています。BigQuery はクエリのスキャン量に対して課金されるので、**dbt run でもコストが発生する**というのは DuckDB + dbt にはない注意点です。

:::message
BigQuery の無料枠は毎月 1TB のクエリスキャン量まで無料なので、チュートリアル程度のデータ量なら心配いりません。
:::

![BigQueryコンソールのテーブル一覧](/images/bigquery-dbt-pipeline/bq-dbt-table-list.png)
*dbt run で作成された staging（ビュー）と mart（テーブル）が確認できる*

## Step 5: テストの追加

モデルの品質チェック用にテストを設定します。

```yaml:models/staging/_schema.yml
version: 2

models:
  - name: stg_users
    columns:
      - name: user_id
        tests:
          - not_null
          - unique
      - name: email
        tests:
          - not_null
          - unique

  - name: stg_orders
    columns:
      - name: order_id
        tests:
          - not_null
          - unique
      - name: user_id
        tests:
          - not_null
          - relationships:
              arguments:
                to: ref('stg_users')
                field: user_id
```

```yaml:models/mart/_schema.yml
version: 2

models:
  - name: mart_user_orders
    columns:
      - name: user_id
        tests:
          - not_null
          - unique
```

`relationships` テストは orders の user_id が全て users に存在するかをチェックするものです。

ここで1つハマりポイントがありました。最初は `relationships` の引数を従来の書き方で書いていたのですが、dbt 1.11 では deprecation warning が出ました。

```yaml
# ❌ 旧記法（deprecation warning が出る）
- relationships:
    to: ref('stg_users')
    field: user_id

# ✅ 新記法（dbt 1.11+）
- relationships:
    arguments:
      to: ref('stg_users')
      field: user_id
```

`arguments` の下にネストする書き方に変更されています。

```bash
uv run dbt test
```

```
1 of 10 PASS not_null_mart_user_orders_user_id .......... [PASS in 1.31s]
2 of 10 PASS not_null_stg_orders_order_id ............... [PASS in 1.20s]
3 of 10 PASS not_null_stg_orders_user_id ................ [PASS in 1.22s]
4 of 10 PASS not_null_stg_users_email ................... [PASS in 1.32s]
5 of 10 PASS not_null_stg_users_user_id ................. [PASS in 1.35s]
6 of 10 PASS relationships_stg_orders_user_id ........... [PASS in 1.27s]
7 of 10 PASS unique_mart_user_orders_user_id ............ [PASS in 0.95s]
8 of 10 PASS unique_stg_orders_order_id ................. [PASS in 1.48s]
9 of 10 PASS unique_stg_users_email ..................... [PASS in 0.95s]
10 of 10 PASS unique_stg_users_user_id .................. [PASS in 0.83s]

Done. PASS=10 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=10
```

10テスト全 PASS。テストの構文自体は DuckDB + dbt と同じですが、BigQuery では各テストがクエリジョブとして実行されるため、テスト数が増えるとスキャン量も増えます。

## Step 6: パーティショニングとクラスタリング

ここからが **BigQuery + dbt ならでは**の部分です。BigQuery 入門ではコンソールや DDL でパーティショニングを設定しましたが、dbt では `config()` でコードとして管理できます。

### seed でサンプルデータを投入

既存テーブルにはタイムスタンプカラムがなかったので、`dbt seed` でパーティショニング検証用のデータを用意しました。

```csv:seeds/daily_access_summary.csv
access_date,user_id,page_path,page_views,unique_visitors
2025-01-01,1,/top,120,80
2025-01-01,2,/live,95,60
2025-01-01,3,/mypage,40,30
2025-01-15,1,/top,130,85
2025-01-15,2,/live,110,70
2025-02-01,1,/top,140,90
2025-02-01,3,/live,100,65
2025-02-15,2,/top,125,82
2025-03-01,1,/top,150,95
2025-03-01,2,/live,115,72
2025-03-01,3,/mypage,55,38
2025-03-15,1,/live,105,68
```

```bash
uv run dbt seed
```

```
1 of 1 OK loaded seed file bq_tutorial.daily_access_summary .. [INSERT 12 in 3.94s]
```

:::message
`dbt seed` は今回のような検証用途やマスターデータ（都道府県コード、カテゴリ定義など）の投入には便利ですが、実運用では 1MB 程度の小さなデータに留めるのがお作法です。大量データの投入には GCS 経由のロードなど別の手段を使います。
:::

### パーティション + クラスタリング付きモデル

`config()` に `partition_by` と `cluster_by` を指定します。

```sql:models/mart/mart_daily_access.sql
{{
    config(
        materialized='table',
        partition_by={
            "field": "access_date",
            "data_type": "date",
            "granularity": "month"
        },
        cluster_by=["page_path"]
    )
}}

with source as (
    select * from {{ ref('daily_access_summary') }}
)

select
    cast(access_date as date) as access_date,
    user_id,
    page_path,
    page_views,
    unique_visitors
from source
```

`partition_by` では `field`（パーティションキー）、`data_type`（データ型）、`granularity`（粒度: day / month / year）を指定します。`cluster_by` はリスト形式で最大4カラムまで指定可能です。

これらは **dbt-bigquery アダプター固有の config** で、DuckDB にはこの概念自体がありません。

```bash
uv run dbt run -s mart_daily_access
```

```
1 of 1 OK created sql table model bq_tutorial.mart_daily_access . [CREATE TABLE (12.0 rows, 467.0 Bytes processed) in 2.94s]
```

BigQuery コンソールでテーブルの詳細を確認すると、ちゃんとパーティションとクラスタリングが設定されていました。

![パーティション設定の確認](/images/bigquery-dbt-pipeline/bq-dbt-partition-detail.png)
*テーブルタイプ「分割」、分割基準「MONTH」、フィールド「access_date」が確認できる*

![クラスタリング設定の確認](/images/bigquery-dbt-pipeline/bq-dbt-cluster-detail.png)
*クラスタ化の基準に「page_path」が設定されている*

| 設定 | 値 |
|---|---|
| テーブルタイプ | 分割 |
| 分割基準 | MONTH |
| フィールドで分割 | access_date |
| クラスタ化の基準 | page_path |

DDL を書かなくても、dbt の config だけでパーティション設計をコード管理できるのは便利です。

## Step 7: Incremental モデル

最後に `incremental`（差分更新）を試しました。`table` は毎回全データを作り直しますが、`incremental` は新しいデータだけを追加します。大規模データの本番運用で使うパターンです。

```sql:models/mart/mart_daily_access_incremental.sql
{{
    config(
        materialized='incremental',
        partition_by={
            "field": "access_date",
            "data_type": "date",
            "granularity": "month"
        },
        cluster_by=["page_path"],
        incremental_strategy='insert_overwrite'
    )
}}

with source as (
    select * from {{ ref('daily_access_summary') }}
)

select
    cast(access_date as date) as access_date,
    user_id,
    page_path,
    page_views,
    unique_visitors
from source

{% if is_incremental() %}
where access_date > (select max(access_date) from {{ this }})
{% endif %}
```

`is_incremental()` は、テーブルが既に存在し `--full-refresh` なしで実行された場合に `true` を返します。つまり初回は全データ INSERT、2回目以降は `where` 句で差分のみ処理されます。

`incremental_strategy='insert_overwrite'` は BigQuery 固有の戦略で、パーティション単位でデータを上書きします。BigQuery で使える戦略を整理すると：

| 戦略 | 動作 | 用途 |
|---|---|---|
| `append` | 単純に行を追加 | 重複を許容するログデータ |
| `insert_overwrite` | パーティション単位で上書き | 日次バッチ処理（推奨） |
| `merge` | MERGE 文で upsert | 主キーベースの更新 |

:::message alert
`insert_overwrite` は `partition_by` の設定が必須です。パーティションが未設定だとエラーになります。パーティション単位で「削除 → 再挿入」する仕組みなので、パーティションキーがないと動作しません。
:::

### 実行結果

**1回目（初回）:**

```bash
uv run dbt run -s mart_daily_access_incremental
```

```
1 of 1 OK created sql incremental model bq_tutorial.mart_daily_access_incremental
  [CREATE TABLE (12.0 rows, 467.0 Bytes processed) in 3.05s]
```

初回は `CREATE TABLE` で全12行が入りました。

**2回目（差分なし）:**

```bash
uv run dbt run -s mart_daily_access_incremental
```

```
1 of 1 OK created sql incremental model bq_tutorial.mart_daily_access_incremental
  [SCRIPT (563.0 Bytes processed) in 6.26s]
```

2回目は `SCRIPT` 表示で行数なし。ソースに新しいデータがないため、差分チェック → 該当0件 → 何も書き込まれない、という動作です。

| | 1回目 | 2回目 |
|---|---|---|
| ログ | `CREATE TABLE (12.0 rows)` | `SCRIPT (563.0 Bytes processed)` |
| 挙動 | 全データ INSERT | 差分0件（書き込みなし） |

本番では日次で新しいデータが追加されるテーブルにこのパターンを適用します。毎回フルスキャンの `table` と比べてスキャン量を大幅に削減できるので、コスト面で効いてきます。

## DuckDB + dbt と BigQuery + dbt の比較

全体を通して感じた違いをまとめます。

### 環境構築

| | DuckDB + dbt | BigQuery + dbt |
|---|---|---|
| 環境 | Docker + requirements.txt | uv + Python 3.12 |
| アダプター | dbt-duckdb | dbt-bigquery |
| 認証 | 不要 | サービスアカウント or OAuth |
| 接続テスト | ファイル存在チェック | GCP API 実接続 |
| location 指定 | 不要 | 必須 |

### 開発体験

| | DuckDB + dbt | BigQuery + dbt |
|---|---|---|
| 実行速度 | 高速（ローカル処理） | ネットワーク往復あり（1クエリ1秒〜） |
| コスト | 無料 | スキャン量課金 |
| パーティション | なし | `partition_by` で設定 |
| クラスタリング | なし | `cluster_by` で設定 |
| incremental 戦略 | `append` / `merge` | `append` / `insert_overwrite` / `merge` |
| デバッグ | ローカル完結 | BQ コンソールでジョブ確認可能 |

### 使い分け

DuckDB + dbt はローカルでの開発・プロトタイピングに向いていて、コストを気にせず試行錯誤できます。BigQuery + dbt は TB 規模以上の大規模データ処理やチームでの共同作業に向いていて、パーティショニングによるコスト最適化が重要になります。

dbt のモデル定義やテスト構文はアダプターに依存しないので、DuckDB で作ったモデルを BigQuery に移行するのはスムーズでした。BigQuery 固有の設定（認証、パーティション、incremental_strategy）を追加するだけで、ほぼ同じコードが動きます。

## コスト面の注意点

BigQuery + dbt を使ってみて気づいたコスト関連のポイントです。

**`dbt run` でスキャン量課金が発生する**: DuckDB では何回 `dbt run` しても無料でしたが、BigQuery では `CREATE TABLE AS SELECT` のたびにスキャンが走ります。開発中は `dbt run -s model_name` で対象を絞るのが良さそうです。

**`dbt test` でもスキャンが走る**: テストもクエリジョブとして実行されるので、テスト数 × テーブルサイズ分のスキャンが発生します。大規模テーブルに大量のテストを設定する場合は注意が必要です。

**`materialized='view'` はスキャン量ゼロ**: ビューの作成自体にはスキャンが発生しません（参照時に発生）。staging 層を view にしているのはコスト面でも合理的でした。

**incremental でスキャン量を削減**: `table` は毎回フルスキャン、`incremental` は差分のみ。大規模テーブルほど差が出ます。

**`dbt compile` + Dry Run で事前にスキャン量を確認**: `dbt compile` を実行すると `target/compiled/` に実際の SQL が生成されます。これを BigQuery コンソールに貼り付けると、右上にスキャン量の見積もり（Dry Run）が表示されます。大きなテーブルに対する新しいモデルを追加する際は、この手順で事前にコストを確認しておくと安心です。

## 最終的なプロジェクト構成

```
bigquery-dbt/
├── .python-version
├── pyproject.toml
└── bigquery_dbt/
    ├── dbt_project.yml
    ├── models/
    │   ├── staging/
    │   │   ├── _sources.yml
    │   │   ├── _schema.yml
    │   │   ├── stg_users.sql
    │   │   └── stg_orders.sql
    │   └── mart/
    │       ├── _schema.yml
    │       ├── mart_user_orders.sql
    │       ├── mart_daily_access.sql
    │       └── mart_daily_access_incremental.sql
    └── seeds/
        └── daily_access_summary.csv
```

## おわりに

DuckDB + dbt の経験があったおかげで、BigQuery + dbt への移行はスムーズでした。モデル定義やテスト構文はそのまま使えて、BigQuery 固有の設定（認証、パーティション、incremental_strategy）を足すだけで動きます。

一方で、BigQuery はスキャン量課金があるので、パーティション設計や incremental の活用が DuckDB 以上に重要だと感じました。dbt の config でこれらを宣言的に管理できるのはありがたいです。

DuckDB でローカル検証 → BigQuery で本番運用、という流れを考えている方の参考になれば幸いです。

### 関連記事

- [DuckDB + S3 + dbt でローカルデータパイプラインを構築してみた](https://zenn.dev/toshiro3/articles/duckdb-s3-dbt-local-pipeline)
- [【BigQuery入門】DuckDBユーザーが初めてクラウドDWHを触ってみた](https://zenn.dev/toshiro3/articles/bigquery-introduction)
- [【入門】DuckDBとは？MySQL/PostgreSQLとの違いを手を動かして理解する](https://zenn.dev/toshiro3/articles/duckdb-introduction)
