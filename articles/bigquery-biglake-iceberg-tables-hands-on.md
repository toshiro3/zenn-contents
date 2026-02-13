---
title: "BigQuery × Apache Iceberg 入門 ― BigLake Icebergテーブルを SQLだけで検証する"
emoji: "🧊"
type: "tech"
topics: ["BigQuery", "ApacheIceberg", "GoogleCloud", "BigLake", "データエンジニアリング"]
published: true
---

## はじめに

この記事では、**BigQuery上でApache Icebergテーブルを作成・操作する「BigLake tables for Apache Iceberg in BigQuery」**（以下、BigLake Icebergテーブル）を実際に手を動かして検証します。

以前の記事でBigQuery + GCS外部テーブル（Parquet / Hiveパーティション）を検証しましたが、外部テーブルには「読み取り専用」「スキーマ変更に弱い」「Time Travelが使えない」といった制約がありました。BigLake Icebergテーブルを使うことで、これらの制約がどのように解消されるかを確認していきます。

### この記事で検証すること

- BigLake Icebergテーブルの作成（DDL、Cloud Resource Connection設定）
- データの書き込み（INSERT / UPDATE / DELETE / MERGE）
- GCS上に生成されるIcebergメタデータの確認
- Time Travel（過去のスナップショットへのクエリ）
- Schema Evolution（カラム追加）とTime Travelの組み合わせ
- GCS外部テーブルとの比較

### 前提知識

- BigQueryの基本操作（SQLの実行、データセットの作成）
- Apache Icebergの基本概念（スナップショット、メタデータ管理）
- GCSの基本操作

:::message
この記事のSQLは、プロジェクトIDを `my-bq-project` としています。実際に試す際はご自身のプロジェクトIDに置き換えてください。
:::

## BigLake Icebergテーブルとは

BigLake Icebergテーブルは、BigQueryのフルマネージド体験を維持しつつ、データをユーザー所有のGCSバケットにIceberg形式で保存するテーブルです。

通常のBigQueryネイティブテーブルとの違いは「データの保存先がGCS」という点で、SQLの書き方やDML操作はほぼ同じです。一方で、Icebergフォーマットで保存されるため、Spark、Trino、Flinkなどの外部エンジンからもデータにアクセスできるというオープン性を持っています。

### BigLake Metastoreの2つの構成パターン

BigLake MetastoreにはIcebergテーブルを管理する2つの方法があります。

**パターン1: BigQueryマネージド（BigLake Iceberg tables in BigQuery）**

BigQueryのSQLでテーブルの作成・変更・データ操作をすべて行う方式です。メタデータはBigLake Metastoreが内部管理し、GCSにはIceberg V2互換のスナップショットがエクスポートされます。自動コンパクション、自動再クラスタリング、ガベージコレクションなどのテーブル管理もBigQueryが自動で行います。

**パターン2: REST Catalog経由（Iceberg REST Catalog）**

SparkやFlinkなどのOSSエンジンからIceberg REST Catalog APIを通じてテーブルを作成・管理する方式です。BigQueryからは外部テーブルとして読み取りアクセスが可能です。OSSエンジンが主体の環境に向いています。

この記事では**パターン1（BigQueryマネージド）**を検証します。BigQueryのSQLだけで完結するため、最も手軽に試せる構成です。

### 料金体系

BigLake Icebergテーブルに関連する主な料金は以下のとおりです。

| 項目 | 料金（東京リージョン） | 備考 |
|------|------|------|
| BigLake Table Management | $0.153/DCU-hour | ストレージ最適化、CMETA生成・更新 |
| BigLake Metastore Class A操作 | 0〜5,001回: 無料 / 超過: $6.00/100万回 | 書き込み・更新・リスト・作成・設定 |
| BigLake Metastore Class B操作 | 0〜50,001回: 無料 / 超過: $0.90/100万回 | 読み取り・取得・削除 |
| BigLake Metastore メタデータストレージ | 無料枠: 1GiB/月 | 超過分は別途課金 |
| GCSストレージ | 通常のGCS料金 | Autoclassで最適化可能 |
| BigQueryクエリ | 通常のBigQuery料金 | オンデマンドまたはリザベーション |

※ 料金はリージョンにより異なります。最新の料金は[BigLake pricing](https://cloud.google.com/products/biglake/pricing)を確認してください。

個人の検証レベルであればMetastoreの無料枠内に収まることがほとんどです。Table Managementはバックグラウンドのストレージ最適化（コンパクション、クラスタリング、GCなど）とCMETA（BigQueryメタデータ）の生成・更新に対して課金されます。

## 環境準備

### 使用環境

- Google Cloudプロジェクト: `my-bq-project`
- リージョン: `asia-northeast1`（東京）
- GCSバケット: `my-bq-project-iceberg`（新規作成）
- データセット: `iceberg_demo`

### Step 1: GCSバケットの作成

Cloud Storageコンソールからバケットを作成します。BigLake Icebergテーブル用のバケットにはいくつかの推奨設定があります。

- **ロケーションタイプ**: Region → `asia-northeast1`（BigQueryデータセットと同じリージョン）
- **ストレージクラス**: Autoclass（アクセス頻度に応じてクラスを自動調整）
- **アクセス制御**: 均一（Uniform） ※BigLake Icebergの要件
- **公開アクセスの防止**: 有効
- **オブジェクトのバージョニング**: 無効 ※必ず無効にしてください

:::message alert
GCSのオブジェクトバージョニングは必ず無効にしてください。Icebergはテーブル自体のスナップショット管理でデータの履歴を保持しているため、GCSバージョニングが有効だとストレージ料金が二重に膨らむだけでなく、意図しない挙動を招く可能性があります。
:::

:::message alert
バケット名はBigLake Icebergテーブル専用であることが分かる名前にしましょう。BigQuery以外からの書き込みはデータ破損の原因になるため、専用バケットにすることが強く推奨されています。
:::

### Step 2: Cloud Resource Connectionの作成

BigQueryコンソールからConnectionを作成します。

1. BigQueryコンソールのエクスプローラーペインで「**＋追加**」→「**外部データソースへの接続**」
2. 接続タイプ: **Vertex AI リモートモデル、リモート関数、BigLake、Spanner（Cloud リソース）**
   - GCS専用の項目はなく、この長い名前の接続タイプがBigLake用です
3. 接続ID: `iceberg-connection`
4. ロケーションタイプ: リージョン → `asia-northeast1`（東京）
5. 「**接続を作成**」をクリック

作成後に表示される**サービスアカウントID**をメモします。

### Step 3: サービスアカウントへの権限付与

GCSバケットの「権限」タブから、Connectionのサービスアカウントに以下の2つのロールを付与します。

- **Storage Object User**（`roles/storage.objectUser`）: オブジェクトの読み書き
- **Storage Legacy Bucket Reader**（`roles/storage.legacyBucketReader`）: バケットメタデータの読み取り

これで環境準備は完了です。

## テーブルの作成

### データセットの作成

```sql
CREATE SCHEMA IF NOT EXISTS `my-bq-project.iceberg_demo`
OPTIONS (
  location = 'asia-northeast1'
);
```

### BigLake Icebergテーブルの作成

```sql
CREATE TABLE `my-bq-project.iceberg_demo.users` (
  user_id INT64 NOT NULL,
  name STRING,
  email STRING,
  age INT64,
  created_at TIMESTAMP
)
CLUSTER BY user_id
WITH CONNECTION `my-bq-project.asia-northeast1.iceberg-connection`
OPTIONS (
  storage_uri = 'gs://my-bq-project-iceberg/iceberg_demo/users',
  file_format = 'PARQUET',
  table_format = 'ICEBERG'
);
```

通常のBigQueryテーブルとの違いは以下の3点だけです。

- `WITH CONNECTION`: Cloud Resource Connectionを指定
- `storage_uri`: GCS上のデータ保存先パス（空のパスを指定する必要あり）
- `file_format = 'PARQUET'` と `table_format = 'ICEBERG'`: ファイル形式とテーブル形式を明示

`CLUSTER BY` も通常のBigQueryテーブルと同様に指定でき、自動再クラスタリングが実行されます。

`INFORMATION_SCHEMA.TABLES` でテーブル情報を確認すると、`table_type` が `BASE TABLE` と表示されます。従来のBigLake外部テーブル（Parquet/Avro等）は `EXTERNAL` として扱われますが、BigLake Icebergテーブルは `BASE TABLE` です。これはBigQueryがメタデータを完全に所有し、ネイティブテーブルと同等の管理権限を持っていることを意味します。

```sql
SELECT table_name, table_type, ddl
FROM `my-bq-project.iceberg_demo.INFORMATION_SCHEMA.TABLES`
WHERE table_name = 'users';
```

| table_name | table_type |
|------------|------------|
| users | BASE TABLE |

## データ操作（DML）

### INSERT

```sql
INSERT INTO `my-bq-project.iceberg_demo.users` (user_id, name, email, age, created_at)
VALUES
  (1, '田中太郎', 'tanaka@example.com', 30, TIMESTAMP '2024-01-15 09:00:00 UTC'),
  (2, '佐藤花子', 'sato@example.com', 25, TIMESTAMP '2024-02-20 10:30:00 UTC'),
  (3, '鈴木一郎', 'suzuki@example.com', 35, TIMESTAMP '2024-03-10 14:00:00 UTC'),
  (4, '高橋美咲', 'takahashi@example.com', 28, TIMESTAMP '2024-04-05 11:15:00 UTC'),
  (5, '伊藤健太', 'ito@example.com', 42, TIMESTAMP '2024-05-12 16:45:00 UTC');
```

```sql
SELECT * FROM `my-bq-project.iceberg_demo.users` ORDER BY user_id;
```

| user_id | name | email | age | created_at |
|---------|------|-------|-----|------------|
| 1 | 田中太郎 | tanaka@example.com | 30 | 2024-01-15 09:00:00 UTC |
| 2 | 佐藤花子 | sato@example.com | 25 | 2024-02-20 10:30:00 UTC |
| 3 | 鈴木一郎 | suzuki@example.com | 35 | 2024-03-10 14:00:00 UTC |
| 4 | 高橋美咲 | takahashi@example.com | 28 | 2024-04-05 11:15:00 UTC |
| 5 | 伊藤健太 | ito@example.com | 42 | 2024-05-12 16:45:00 UTC |

INSERT文も通常のBigQueryテーブルと全く同じ構文で動作します。

### UPDATE

```sql
UPDATE `my-bq-project.iceberg_demo.users`
SET age = 31, name = '田中太郎（更新）'
WHERE user_id = 1;
```

### DELETE

```sql
DELETE FROM `my-bq-project.iceberg_demo.users`
WHERE user_id = 6;
```

### MERGE（UPSERT）

既存データの更新と新規データの挿入を1文で実行できます。

```sql
MERGE INTO `my-bq-project.iceberg_demo.users` AS target
USING (
  SELECT 1 AS user_id, '田中太郎' AS name, 'tanaka_new@example.com' AS email,
         32 AS age, TIMESTAMP '2024-01-15 09:00:00 UTC' AS created_at, '神奈川県' AS prefecture
  UNION ALL
  SELECT 7, '渡辺七海', 'watanabe@example.com',
         29, TIMESTAMP '2024-07-20 13:00:00 UTC', '福岡県'
) AS source
ON target.user_id = source.user_id
WHEN MATCHED THEN
  UPDATE SET name = source.name, email = source.email,
             age = source.age, prefecture = source.prefecture
WHEN NOT MATCHED THEN
  INSERT (user_id, name, email, age, created_at, prefecture)
  VALUES (source.user_id, source.name, source.email,
          source.age, source.created_at, source.prefecture);
```

INSERT / UPDATE / DELETE / MERGEのすべてが、BigQueryネイティブテーブルと同じ構文で動作します。GCS外部テーブル（読み取り専用）との大きな違いです。

:::message
BigLake Icebergテーブルでは、同じテーブルに対する複数の書き込み操作（UPDATE / DELETE / MERGEなどの変更DMLやロード）が競合した場合、BigQueryが自動的にキューイング（待ち行列）に入れて直列化して実行します。エラーになるわけではなく順番待ちになるため、リトライ処理を実装する必要はありません。INSERT文にはこの制限はありません。
:::

## GCS上のIcebergメタデータ確認

BigLake Icebergテーブルにデータを投入した後、GCSバケットを確認すると以下のようなファイル構造が生成されています。

```
my-bq-project-iceberg/
  └── iceberg_demo/users/
        ├── data/
        │     └── 8388953c-c922-442e-9c57-XXXX.parquet  (1.9 KB)
        └── metadata/
              └── v0.metadata.json  (97 B)
```

### data/ フォルダ

Parquet形式のデータファイルが格納されます。DML操作を行うたびに新しいParquetファイルが追加されます（Icebergのcopy-on-write方式）。ファイル名はUUID形式です。

INSERT → UPDATE → INSERT と3回のDML操作を行った後は、dataフォルダ内に3つのParquetファイルが生成されていました。

```
data/
  ├── 425dec54-3d1c-49f0-XXXX.parquet  (1.7 KB)  ← UPDATE時に生成
  ├── 8388953c-c922-442e-XXXX.parquet  (1.9 KB)  ← 初回INSERT時に生成
  └── 874a60b3-5eca-4c69-XXXX.parquet  (2.2 KB)  ← 追加INSERT時に生成
```

### metadata/ フォルダ

ここが通常のIceberg（Trino / Spark管理）との大きな違いです。

通常のIcebergでは、`metadata/` フォルダに `v1.metadata.json`, `v2.metadata.json`, `snap-*.avro`, `manifest-*.avro` といった多数のファイルが生成されます。しかし、BigLake Icebergテーブルでは **`v0.metadata.json`（97B）が1つだけ** でした。

これは、BigLake Icebergテーブルのメタデータ管理がBigLake Metastoreの内部で行われているためです。GCS上のメタデータはあくまでIceberg V2互換のエクスポート用であり、メタデータの実体はBigLake Metastoreが管理しています。INSERT / UPDATE / ALTER TABLEを何度実行しても、GCS上のmetadataフォルダの内容は変化しませんでした。

:::message
通常のIcebergではメタデータファイルの増大がパフォーマンスに影響することがありますが、BigLake Icebergテーブルではその心配がありません。Google内部のスケーラブルなメタデータ管理システムが使われているためです。
:::

## Time Travel

BigLake Icebergテーブルでは、BigQueryネイティブテーブルと同じ `FOR SYSTEM_TIME AS OF` 構文でTime Travelが可能です。

### 検証手順

1. 最初に5件のデータをINSERT（この時点のスナップショットが作成される）
2. user_id=1のデータをUPDATE（`田中太郎` → `田中太郎（更新）`、age: 30 → 31）
3. user_id=6のデータを追加INSERT

### 現在のデータを確認

```sql
SELECT user_id, name, age
FROM `my-bq-project.iceberg_demo.users`
WHERE user_id = 1;
```

| user_id | name | age |
|---------|------|-----|
| 1 | 田中太郎（更新） | 31 |

### UPDATE前の時点にTime Travel

```sql
SELECT user_id, name, age
FROM `my-bq-project.iceberg_demo.users`
  FOR SYSTEM_TIME AS OF TIMESTAMP '2026-02-14T01:50:00+09:00'
WHERE user_id = 1;
```

| user_id | name | age |
|---------|------|-----|
| 1 | 田中太郎 | 30 |

UPDATE前の状態（`田中太郎`、age=30）が取得できました。GCS外部テーブルではTime Travelは使えないため、これはBigLake Icebergテーブルの大きなメリットです。

## Schema Evolution

### カラムの追加

```sql
ALTER TABLE `my-bq-project.iceberg_demo.users`
ADD COLUMN prefecture STRING;
```

`ALTER TABLE ... ADD COLUMN` で既存テーブルにカラムを追加できます。既存の行の `prefecture` は `NULL` になります。

```sql
UPDATE `my-bq-project.iceberg_demo.users`
SET prefecture = '東京都'
WHERE user_id IN (1, 2, 3);

UPDATE `my-bq-project.iceberg_demo.users`
SET prefecture = '大阪府'
WHERE user_id IN (4, 5, 6);
```

```sql
SELECT user_id, name, age, prefecture
FROM `my-bq-project.iceberg_demo.users`
ORDER BY user_id;
```

| user_id | name | age | prefecture |
|---------|------|-----|------------|
| 1 | 田中太郎（更新） | 31 | 東京都 |
| 2 | 佐藤花子 | 25 | 東京都 |
| 3 | 鈴木一郎 | 35 | 東京都 |
| 4 | 高橋美咲 | 28 | 大阪府 |
| 5 | 伊藤健太 | 42 | 大阪府 |
| 6 | 山田次郎 | 33 | 大阪府 |

### Schema Evolution × Time Travel

カラム追加前の時点にTime Travelすると、スキーマごと巻き戻ります。

```sql
SELECT *
FROM `my-bq-project.iceberg_demo.users`
  FOR SYSTEM_TIME AS OF TIMESTAMP '2026-02-14T01:50:00+09:00'
ORDER BY user_id;
```

| user_id | name | email | age | created_at |
|---------|------|-------|-----|------------|
| 1 | 田中太郎 | tanaka@example.com | 30 | 2024-01-15 09:00:00 UTC |
| 2 | 佐藤花子 | sato@example.com | 25 | 2024-02-20 10:30:00 UTC |
| 3 | 鈴木一郎 | suzuki@example.com | 35 | 2024-03-10 14:00:00 UTC |
| 4 | 高橋美咲 | takahashi@example.com | 28 | 2024-04-05 11:15:00 UTC |
| 5 | 伊藤健太 | ito@example.com | 42 | 2024-05-12 16:45:00 UTC |

注目すべきポイントが3つあります。

1. **`prefecture` カラムが表示されない** — Time Travelはスキーマも含めてその時点の状態に戻る
2. **行数が5件** — user_id=6（後から追加したデータ）は含まれない
3. **user_id=1が `田中太郎` / age=30** — UPDATEも巻き戻っている

Icebergのスナップショットはスキーマ情報も含めて管理されているため、Time Travelでスキーマごと過去の状態に戻れます。これはIcebergのSchema Evolutionとスナップショット管理が密接に連携しているからこそ可能な挙動です。

## GCS外部テーブルとの比較

以前検証したGCS外部テーブル（Parquet + Hiveパーティション）と、BigLake Icebergテーブルを比較します。

| 機能 | GCS外部テーブル | BigLake Icebergテーブル |
|------|----------------|------------------------|
| データの書き込み | ❌ 読み取り専用 | ✅ INSERT / UPDATE / DELETE / MERGE |
| Time Travel | ❌ 不可 | ✅ `FOR SYSTEM_TIME AS OF` |
| Schema Evolution | ❌ 外部で対応が必要 | ✅ `ALTER TABLE ADD COLUMN` |
| 自動コンパクション | ❌ 手動対応 | ✅ BigQueryが自動管理 |
| 自動クラスタリング | ❌ なし | ✅ `CLUSTER BY` + 自動再クラスタリング |
| ガベージコレクション | ❌ 手動対応 | ✅ BigQueryが自動管理 |
| MERGE（UPSERT） | ❌ 不可 | ✅ 対応 |
| メタデータ管理 | 手動（ファイルパス指定） | BigLake Metastoreが自動管理 |
| OSSエンジンからのアクセス | ✅ 直接読み取り | ✅ Iceberg V2スナップショット経由 |
| テーブルタイプ | EXTERNAL | BASE TABLE |
| GCS上のファイル形式 | Parquet | Parquet（Iceberg管理） |
| table_type | EXTERNAL | BASE TABLE |

BigLake Icebergテーブルは、GCS外部テーブルの「GCSにデータがある」というオープン性を維持しながら、BigQueryネイティブテーブルに近い操作性を実現しています。

## 自動テーブル管理

BigLake Icebergテーブルでは、以下のテーブル管理がBigQueryによって自動的に行われます。手動で `OPTIMIZE` や `VACUUM` を実行する必要はありません。

- **適応的ファイルサイジング**: テーブルサイズに基づいて最適なファイルサイズを自動決定
- **自動コンパクション**: 小さなファイルを最適なサイズに結合
- **自動再クラスタリング**: `CLUSTER BY` で指定したカラムに基づいてデータを再配置
- **ガベージコレクション**: 不要になったデータファイルをTime Travel期間後に自動削除

今回の検証では少量データのため自動コンパクションの動作は確認できませんでしたが、DML操作のたびにParquetファイルが増えていく様子は確認でき、大規模データでは自動コンパクションが効いてくることが想像できます。

## クリーンアップ

検証が終わったら以下で削除します。

```sql
-- テーブル削除
DROP TABLE IF EXISTS `my-bq-project.iceberg_demo.users`;

-- データセット削除
DROP SCHEMA IF EXISTS `my-bq-project.iceberg_demo` CASCADE;
```

テーブルを削除した後も、**GCS上のデータファイルはすぐには削除されません**。Time Travel期間の満了後にバックグラウンドのガベージコレクションによって自動的に削除されます。

:::message
BigQueryのTime Travel期間はデフォルトで7日間ですが、データセットの設定で2日〜7日の間で変更可能です。短く設定するとストレージコストの節約になりますが、その分過去のスナップショットへのアクセス期間も短くなります。
:::

すぐに完全削除したい場合は、Connectionとバケットもコンソールから手動で削除してください。

1. BigQueryコンソール → 接続 → `iceberg-connection` → 削除
2. Cloud Storageコンソール → `my-bq-project-iceberg` → 削除

## まとめ

BigLake Icebergテーブルを検証した結果、以下のことが分かりました。

**良かった点**

- DDLやDMLの構文がBigQueryネイティブテーブルとほぼ同じで、学習コストが低い
- INSERT / UPDATE / DELETE / MERGEがすべて使えるため、GCS外部テーブルの「読み取り専用」制約が解消される
- Time TravelやSchema Evolutionがスナップショット管理と連携して動作する
- 自動コンパクション、自動クラスタリング、GCなどのテーブル管理が不要（`OPTIMIZE` / `VACUUM` 不要）
- メタデータ管理がBigLake Metastore内部で行われるため、GCS上のメタデータファイル増大を気にしなくてよい

**注意が必要な点**

- 変更DML（UPDATE / DELETE / MERGE）は同じテーブルに対してキューイングされ直列実行される（失敗はしないが待ちが発生する）
- GCS上のファイルをBigQuery以外から変更するとデータ破損のリスクがある
- テーブルのコピー、クローン、リネームなどの操作は非対応
- バケットは専用にすることが強く推奨されている

BigQuery + GCS外部テーブルの「データの所在がオープン」というメリットを活かしつつ、BigQueryネイティブテーブルに近い操作性を得られるBigLake Icebergテーブルは、Google Cloud上でオープンレイクハウスを構築する際の有力な選択肢です。

## 参考リンク

- [Use BigLake tables for Apache Iceberg in BigQuery（公式ドキュメント）](https://cloud.google.com/bigquery/docs/iceberg-tables)
- [Introduction to BigLake metastore（公式ドキュメント）](https://docs.cloud.google.com/biglake/docs/about-blms)
- [BigLake pricing（料金）](https://cloud.google.com/products/biglake/pricing)
- [Enhancing BigLake for Iceberg lakehouses（Google Cloud Blog）](https://cloud.google.com/blog/products/data-analytics/enhancing-biglake-for-iceberg-lakehouses)
