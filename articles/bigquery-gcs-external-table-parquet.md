---
title: "BigQuery × GCS外部テーブルでParquetファイルをそのままクエリしてみた"
emoji: "🔍"
type: "tech"
topics: ["BigQuery", "GoogleCloudStorage", "GCS", "Parquet", "DataEngineering"]
published: true
---

## はじめに

BigQueryにはGCS（Google Cloud Storage）上のファイルを直接クエリできる「外部テーブル」という機能があります。データをBigQueryにロードせず、GCSに置いたParquetやCSVをそのままSQLで分析できます。

以前、[DuckDB + S3 + dbtでローカルデータパイプラインを構築する記事](https://zenn.dev/toshiro3/articles/duckdb-s3-dbt-local-pipeline)を書きましたが、今回はそのBigQuery版として、GCS外部テーブルを一通り検証しました。外部テーブルの作成からHiveパーティション対応、ネイティブテーブルとの性能比較まで試し、最後にDuckDB + S3との違いも整理しています。

検証コードはGitHubリポジトリに置いています。

https://github.com/toshiro3/bq-gcs-external

## 外部テーブルとは

BigQueryのテーブルには大きく2種類あります。

**ネイティブテーブル**はBigQueryの内部ストレージにデータを格納する通常のテーブルで、BigQueryの列指向フォーマット（Capacitor）で最適化されて保存されるため、クエリ性能が高いです。

**外部テーブル**はデータの実体がGCSやGoogle Drive等の外部ストレージにあり、BigQuery側にはメタデータ（スキーマやファイルの場所）だけが登録されます。クエリ実行時に外部ストレージからデータを読み取る仕組みです。

外部テーブルのメリットは、GCSにあるデータをBigQueryにロードする手間なくSQLでクエリできること、ストレージの二重持ちを避けられることが挙げられます。一方、クエリのたびに外部ストレージへの読み取りが発生するため、ネイティブテーブルに比べてパフォーマンスが落ちる点、クエリ結果のキャッシュが効かない点がデメリットになります。

## 検証環境

今回の検証環境は以下の通りです。

- Google Cloudプロジェクト: `my-bq-project`
- BigQueryデータセット: `bq_tutorial`（`asia-northeast1`／東京リージョン）
- GCSバケット: `my-bq-project-gcs-external`（`asia-northeast1`／東京リージョン）
- Python 3.12、uv（パッケージ管理）
- 使用ライブラリ: pandas、pyarrow、google-cloud-storage、google-cloud-bigquery

プロジェクトはuvで管理しました。

```bash
uv init bq-gcs-external
cd bq-gcs-external
uv python pin 3.12
uv add pandas pyarrow google-cloud-storage google-cloud-bigquery
```

## GCSバケットの作成とファイルアップロード

### バケット作成

まずGCSバケットをGoogle Cloudコンソールから作成しました。設定のポイントはロケーションタイプの選択で、選択肢は以下の3つがあります。

- **Region**: 単一リージョン。最もコストが安く、同一リージョンからのアクセスが最速
- **Dual-region**: 2つのリージョンにデータ複製。可用性が高いがコスト増
- **Multi-region**: 大陸レベルで複製。最も高可用だが最もコスト高

今回はBigQueryデータセットと同じ東京リージョン（`asia-northeast1`）で、Regionを選択しました。BigQueryの外部テーブルはGCSバケットと同一リージョンでないとデータ転送料金が発生したり、クエリ自体がエラーになるケースもあるため、リージョンを揃えるのが基本です。

作成したバケットの設定は以下の通りです。

- 名前: `my-bq-project-gcs-external`
- ロケーション: `asia-northeast1`（東京）
- ストレージクラス: Standard
- 公開アクセス: 非公開

![GCSバケット作成完了](/images/bigquery-gcs-external-table-parquet/gcs-bucket-created.png)

### ダミーデータの生成

サンプルデータとして、usersテーブル（100行）とordersテーブル（1,000行）をPythonで生成し、Parquet形式で出力しました。

```python:src/generate_data.py
import pandas as pd
import numpy as np
from datetime import datetime, timedelta

np.random.seed(42)
n_users = 100

users = pd.DataFrame({
    "user_id": range(1, n_users + 1),
    "name": [f"user_{i:03d}" for i in range(1, n_users + 1)],
    "email": [f"user_{i:03d}@example.com" for i in range(1, n_users + 1)],
    "age": np.random.randint(18, 65, n_users),
    "prefecture": np.random.choice(
        ["東京都", "大阪府", "愛知県", "福岡県", "北海道"], n_users
    ),
    "created_at": [
        datetime(2025, 1, 1) + timedelta(days=int(d))
        for d in np.random.randint(0, 365, n_users)
    ],
})

n_orders = 1000

orders = pd.DataFrame({
    "order_id": range(1, n_orders + 1),
    "user_id": np.random.randint(1, n_users + 1, n_orders),
    "product": np.random.choice(
        ["プランA", "プランB", "プランC", "プランD"], n_orders
    ),
    "amount": np.random.randint(500, 50000, n_orders),
    "order_date": [
        datetime(2025, 1, 1) + timedelta(days=int(d))
        for d in np.random.randint(0, 365, n_orders)
    ],
})

users.to_parquet("data/users.parquet", index=False)
orders.to_parquet("data/orders.parquet", index=False)

print(f"users: {len(users)}行")
print(f"orders: {len(orders)}行")
```

```bash
$ uv run python src/generate_data.py
users: 100行
orders: 1000行
Parquetファイルを出力しました
```

### GCSへのアップロード

生成したParquetファイルをgsutilでGCSにアップロードしました。

```bash
$ gsutil cp data/users.parquet gs://my-bq-project-gcs-external/
Copying file://data/users.parquet [Content-Type=application/octet-stream]...
/ [1 files][  6.5 KiB/  6.5 KiB]
Operation completed over 1 objects/6.5 KiB.

$ gsutil cp data/orders.parquet gs://my-bq-project-gcs-external/
Copying file://data/orders.parquet [Content-Type=application/octet-stream]...
/ [1 files][ 19.0 KiB/ 19.0 KiB]
Operation completed over 1 objects/19.0 KiB.
```

GCSコンソールで確認すると、バケット内にファイルが表示されました。

![GCSにParquetファイルをアップロード](/images/bigquery-gcs-external-table-parquet/gcs-files-uploaded.png)

## 外部テーブルの作成

### DDLで外部テーブルを作成

BigQueryコンソールのクエリエディタで以下のDDLを実行し、外部テーブルを作成しました。

```sql
-- users 外部テーブル
CREATE OR REPLACE EXTERNAL TABLE `bq_tutorial.ext_users`
OPTIONS (
  format = 'PARQUET',
  uris = ['gs://my-bq-project-gcs-external/users.parquet']
);

-- orders 外部テーブル
CREATE OR REPLACE EXTERNAL TABLE `bq_tutorial.ext_orders`
OPTIONS (
  format = 'PARQUET',
  uris = ['gs://my-bq-project-gcs-external/orders.parquet']
);
```

![ext_users 外部テーブル作成](/images/bigquery-gcs-external-table-parquet/create-ext-users.png)
![ext_orders 外部テーブル作成](/images/bigquery-gcs-external-table-parquet/create-ext-orders.png)

Parquetファイルの場合、スキーマ定義は不要で、ファイルのメタデータから自動的にカラム名と型が推論されます。実際にスキーマタブを確認すると、`user_id`がINTEGER、`name`がSTRING、`created_at`がTIMESTAMPといった具合に適切な型が付いていました。CSVの場合はカラム定義を明示的に書く必要があるため、Parquetの方が手軽です。

![ext_users スキーマ（Parquetから自動推論）](/images/bigquery-gcs-external-table-parquet/ext-users-schema.png)
![ext_orders スキーマ（Parquetから自動推論）](/images/bigquery-gcs-external-table-parquet/ext-orders-schema.png)

テーブル名に`ext_`プレフィックスを付けたのは、ネイティブテーブルと区別しやすくするための命名慣習です。

### SELECTで確認

作成した外部テーブルの中身をSELECTで確認しました。

```sql
SELECT * FROM `bq_tutorial.ext_users` LIMIT 10;
```

GCS上のParquetファイルの内容がそのまま返ってきました。

![ext_users のSELECT結果](/images/bigquery-gcs-external-table-parquet/select-ext-users.png)

外部テーブル同士のJOINも問題なく動作します。

```sql
-- ユーザーごとの注文集計（外部テーブル同士のJOIN）
SELECT
  u.name,
  u.prefecture,
  COUNT(o.order_id) AS order_count,
  SUM(o.amount) AS total_amount
FROM `bq_tutorial.ext_users` u
JOIN `bq_tutorial.ext_orders` o ON u.user_id = o.user_id
GROUP BY u.name, u.prefecture
ORDER BY total_amount DESC
LIMIT 10;
```

![外部テーブル同士のJOIN結果](/images/bigquery-gcs-external-table-parquet/join-ext-tables.png)

外部テーブルであっても、通常のテーブルと同じようにSQLが書けます。

## ネイティブテーブルとの性能比較

外部テーブルとネイティブテーブルでどの程度の性能差があるのか確認しました。

### ネイティブテーブルの作成

CTASで外部テーブルからネイティブテーブルを作成しました。

```sql
CREATE OR REPLACE TABLE `bq_tutorial.native_users` AS
SELECT * FROM `bq_tutorial.ext_users`;

CREATE OR REPLACE TABLE `bq_tutorial.native_orders` AS
SELECT * FROM `bq_tutorial.ext_orders`;
```

### 同一クエリでの比較

同じJOINクエリを外部テーブルとネイティブテーブルの両方で実行し、ジョブ情報を比較しました。

| 項目 | 外部テーブル | ネイティブテーブル |
|---|---|---|
| 処理されたバイト数 | 26.27 KB | 26.27 KB |
| 課金されるバイト数 | 20 MB | 20 MB |
| スロット（ミリ秒） | **2,748** | **22** |
| 期間 | 0 秒 | 0 秒 |

処理バイト数と課金バイト数は同じですが、スロット消費時間に約125倍の差が出ました。この差が生まれる背景として、ネイティブテーブルはBigQuery専用の列指向フォーマット（Capacitor）で保存されており、列の読み飛ばしや圧縮効率が極限まで最適化されている点があります。一方、外部テーブルはクエリのたびにBigQueryのCompute層からGCS（Storage層）へネットワーク越しにデータを転送するオーバーヘッドが発生します。ただし、今回のデータは小さいため差が極端に出ている面もあり、外部テーブルが常にこれほど遅いわけではありません。

課金バイト数がどちらも20MBなのは、BigQueryには1テーブルあたり最低10MBの課金下限があるためです。2テーブルをJOINしているので10MB × 2 = 20MBとなっています。実データは26.27KBしかないのに20MB分課金されるという点は注意が必要です。小さなテーブルを多数JOINするクエリでは、このテーブル単位の課金下限が積み重なって意外とコストが膨らむことがあります。

また、外部テーブルのジョブ情報には「メタデータキャッシュが使用されない理由: `METADATA_CACHING_NOT_ENABLED`」と表示されていました。外部テーブルはデフォルトでメタデータキャッシュが無効ですが、`metadata_cache_mode = 'AUTOMATIC'`を設定して有効化すると、ファイル一覧の取得等がキャッシュされてパフォーマンスが改善される可能性があります。

### 使い分けの指針

今回のデータは小さすぎて体感差はありませんでしたが、データ量が増えるほどスロット消費の差は広がります。頻繁にクエリするテーブルはネイティブテーブルへの取り込み（CTASや`LOAD DATA`）を検討すべきです。外部テーブルは「GCSに溜まったデータをとりあえずSQLで確認したい」「ETLパイプラインの中間ステージとして一時的に参照したい」といったユースケースに向いています。

## Hiveパーティション形式の外部テーブル

外部テーブルの実用的な機能として、Hiveパーティション形式への対応があります。ディレクトリ構造をパーティションキーとして認識し、WHERE句でフィルタすると該当パーティションのファイルだけを読み取ります（パーティションプルーニング）。

### パーティション分割データの生成

ordersデータを年月（`year_month`）でパーティション分割し、Hive形式のディレクトリ構造で出力しました。

```python:src/generate_partitioned_data.py
import pandas as pd
import numpy as np
from datetime import datetime, timedelta
from pathlib import Path

np.random.seed(42)
n_orders = 1000

orders = pd.DataFrame({
    "order_id": range(1, n_orders + 1),
    "user_id": np.random.randint(1, 101, n_orders),
    "product": np.random.choice(
        ["プランA", "プランB", "プランC", "プランD"], n_orders
    ),
    "amount": np.random.randint(500, 50000, n_orders),
    "order_date": [
        datetime(2025, 1, 1) + timedelta(days=int(d))
        for d in np.random.randint(0, 365, n_orders)
    ],
})

orders["year_month"] = orders["order_date"].dt.strftime("%Y-%m")

output_dir = Path("data/partitioned_orders")
output_dir.mkdir(parents=True, exist_ok=True)

for ym, group in orders.groupby("year_month"):
    partition_dir = output_dir / f"year_month={ym}"
    partition_dir.mkdir(exist_ok=True)
    group.drop(columns=["year_month"]).to_parquet(
        partition_dir / "data.parquet", index=False
    )
    print(f"  {ym}: {len(group)}行")
```

```bash
$ uv run python src/generate_partitioned_data.py
  2025-01: 73行
  2025-02: 75行
  2025-03: 90行
  ...
  2025-12: 91行
```

出力されるディレクトリ構造はこのようになります。`year_month=2025-01`のようなディレクトリ名がHive形式のパーティションキーです。

```
data/partitioned_orders/
├── year_month=2025-01/
│   └── data.parquet
├── year_month=2025-02/
│   └── data.parquet
├── ...
└── year_month=2025-12/
    └── data.parquet
```

### GCSへのアップロードと外部テーブル作成

gsutilの`-m`オプションで並列アップロードしました。

```bash
$ gsutil -m cp -r data/partitioned_orders gs://my-bq-project-gcs-external/
/ [12/12 files][ 58.7 KiB/ 58.7 KiB] 100% Done
Operation completed over 12 objects/58.7 KiB.
```

BigQueryコンソールでHiveパーティション対応の外部テーブルを作成しました。

```sql
CREATE OR REPLACE EXTERNAL TABLE `bq_tutorial.ext_orders_partitioned`
WITH PARTITION COLUMNS (
  year_month STRING
)
OPTIONS (
  format = 'PARQUET',
  uris = ['gs://my-bq-project-gcs-external/partitioned_orders/*'],
  hive_partition_uri_prefix = 'gs://my-bq-project-gcs-external/partitioned_orders/'
);
```

![Hiveパーティション外部テーブル作成](/images/bigquery-gcs-external-table-parquet/create-ext-orders-partitioned.png)

通常の外部テーブルとの違いは以下の3点です。

- `WITH PARTITION COLUMNS`でパーティションカラムの名前と型を明示する
- `uris`にはワイルドカード（`*`）を使い、全パーティション配下のファイルを対象にする
- `hive_partition_uri_prefix`でパーティションのベースパスを指定する

なお、`WITH PARTITION COLUMNS`を省略してBigQueryにパーティションカラムの型を自動推論させることも可能です。ただし、意図しない型推論（例：日付文字列が別の型になる等）を避けるために、明示的に定義しておく方が安全です。

### パーティションプルーニングの検証

パーティションプルーニングが効くか確認するために、フィルタなしのクエリと特定月のみのクエリを実行して比較しました。

```sql
-- 全パーティション（フィルタなし）
SELECT COUNT(*), SUM(amount) FROM `bq_tutorial.ext_orders_partitioned`;

-- 特定パーティションのみ
SELECT COUNT(*), SUM(amount) FROM `bq_tutorial.ext_orders_partitioned`
WHERE year_month = '2025-01';
```

実行結果のジョブ情報を比較すると、以下のようになりました。

| 項目 | フィルタなし（全12ヶ月） | WHERE year_month = '2025-01' |
|---|---|---|
| 件数 | 1,000 | 73 |
| 処理されたバイト数 | **7.81 KB** | **584 B** |
| 課金されるバイト数 | 10 MB | 10 MB |
| スロット（ミリ秒） | 672 | 158 |

処理バイト数が7.81KB → 584Bと約13分の1に減少しました。12ヶ月のうち1ヶ月分だけ読んでいるので理論通りの結果です。スロット消費も672 → 158と大幅に削減されています。

課金バイト数は最低10MBの下限に引っかかっているため同じですが、実データが大きければ課金額にも直接効いてきます。データレイク上の大量のログデータを日付でフィルタして分析するような場面では、Hiveパーティションによるプルーニングがコスト削減に直結します。

## DuckDB + S3（httpfs）との比較

以前の記事で検証したDuckDB + S3（httpfs）と、今回のBigQuery + GCS外部テーブルを比較してみます。

### アクセス方法の違い

| 観点 | BigQuery + GCS外部テーブル | DuckDB + S3（httpfs） |
|---|---|---|
| 定義方法 | `CREATE EXTERNAL TABLE`でDDLとして事前定義 | `read_parquet('s3://...')`でクエリ内で直接指定 |
| スキーマ管理 | テーブルとしてカタログに登録される | なし（クエリ時に都度推論） |
| Hiveパーティション | `WITH PARTITION COLUMNS` + `hive_partition_uri_prefix` | `hive_partitioning=true`オプション |
| ワイルドカード | `uris`にワイルドカード指定 | globパターン（`*.parquet`） |

BigQueryは外部テーブルとして事前にDDLで定義する形式です。テーブルとしてカタログに登録されるため、他のユーザーからも同じテーブル名でクエリできます。一方DuckDBは`read_parquet()`関数でクエリ内に直接パスを指定する形で、テーブル定義なしにすぐクエリできる手軽さがあります。

Hiveパーティションの扱いも対照的で、BigQueryは`WITH PARTITION COLUMNS`でカラム名と型をDDLで明示的に定義するのに対し、DuckDBは`hive_partitioning=true`というオプションを渡すだけで自動認識します。

### コスト構造の違い

| 観点 | BigQuery + GCS | DuckDB + S3 |
|---|---|---|
| 実行環境 | フルマネージド（サーバーレス） | ローカルマシン or サーバー |
| 課金体系 | スキャン量課金（最低10MB/テーブル） + GCSアクセス料金 | S3リクエスト料金のみ（コンピュートは自前） |
| 同時実行 | 複数ユーザーが同時クエリ可能 | プロセス単位 |
| セットアップ | GCPプロジェクト + IAM設定 | `pip install duckdb`で即開始 |

BigQueryはフルマネージドでインフラ管理が不要な代わりに、スキャン量に応じた課金が発生します。DuckDBはローカルで動くため課金はS3のリクエスト料金程度ですが、コンピュートリソースは自前で用意する必要があります。

### ユースケースの使い分け

| ユースケース | おすすめ |
|---|---|
| チームで共有するデータ分析基盤 | BigQuery + GCS |
| 個人のアドホック分析・探索 | DuckDB + S3 |
| データレイクの定期クエリ | BigQuery（外部テーブル → ネイティブ化も検討） |
| ローカルでの高速な前処理・検証 | DuckDB |
| GCS/S3に溜まったログの一時的な確認 | どちらもOK（手軽さならDuckDB） |

どちらが優れているという話ではなく、チームでの共有や本番パイプラインにはBigQuery、個人の探索的分析やローカルでの検証にはDuckDB、という使い分けになります。両方使えるようにしておくと、場面に応じて最適な選択ができます。

## 外部テーブルの制限事項

実運用で外部テーブルを使う場合、いくつか注意すべき制限があります。

**同時実行数の制限**: 外部テーブルに対するクエリは、ネイティブテーブルに比べて同時実行スロットの制限を受けやすい場合があります。多数のユーザーが同時にクエリを投げるようなワークロードでは、外部テーブルがボトルネックになる可能性があります。

**結果整合性**: GCS上のファイルを上書き更新した場合、外部テーブル側への反映が即時ではなく、クエリが一時的に古いデータを返したり失敗したりする可能性があります。ファイルの追加は比較的安全ですが、既存ファイルの上書きや削除を伴う運用では注意が必要です。

## まとめ

BigQuery + GCS外部テーブルを一通り検証しました。振り返ると、以下のことが確認できました。

GCS上のParquetファイルに対して`CREATE EXTERNAL TABLE`で外部テーブルを定義すれば、データをBigQueryにロードすることなくSQLでクエリできます。Parquetの場合はスキーマ定義も不要で、セットアップは非常に手軽です。

ただし、ネイティブテーブルとの比較ではスロット消費時間に約125倍の差が出ました。外部テーブルはGCSからデータを読みに行くオーバーヘッドがあるため、頻繁にクエリするテーブルはネイティブ化を検討した方がよいです。

Hiveパーティション形式の外部テーブルでは、WHERE句によるパーティションプルーニングが正しく動作し、処理バイト数が約13分の1に削減されました。大量データを日付等でフィルタして分析するユースケースでは、パーティション設計がコスト削減に直結します。

DuckDB + S3と比較すると、BigQueryはチームでの共有や本番パイプラインに強く、DuckDBは個人の探索的分析に強いです。ツールの特性を理解して使い分けることが大事だと感じました。
