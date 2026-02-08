---
title: "【BigQuery入門】DuckDBユーザーが初めてクラウドDWHを触ってみた"
emoji: "🔍"
type: "tech"
topics: ["BigQuery", "GCP", "SQL", "DuckDB", "データエンジニアリング"]
published: true
---

## はじめに

これまでDuckDBを使ったローカルでのデータ分析を行ってきましたが、今回はクラウドDWHの世界に踏み出してみることにしました。

最初のターゲットは **Google BigQuery**。サーバーレスで、ペタバイト級のデータを扱えるフルマネージドなデータウェアハウスです。

本記事では、BigQueryのアカウント準備からテーブル作成、SQLクエリ、パーティショニングまで、一通りの基本操作を実際に手を動かして検証した内容をまとめます。

### 対象読者

- DuckDBやMySQL/PostgreSQLの経験があり、初めてBigQueryを触る方
- データエンジニアリングに興味があり、クラウドDWHの基礎を学びたい方
- BigQueryの無料枠で検証を始めたい方

### 検証環境

- Google Cloud プロジェクト: `my-bq-project`
- リージョン: `asia-northeast1`（東京）
- 料金プラン: オンデマンド（無料枠内で検証）

## BigQueryとは

BigQueryはGoogleが提供する **サーバーレスのデータウェアハウス** です。主な特徴は以下の通りです。

**ストレージとコンピュートの分離**
BigQueryの最大の特徴は、データを保存するストレージと、クエリを処理するコンピュートが完全に分離されていることです。これにより、それぞれを独立してスケールでき、クエリを実行していないときはコンピュートのコストがかかりません。

**サーバーレス**
インフラの管理が不要です。サーバーのプロビジョニング、チューニング、パッチ適用といった作業は一切ありません。SQLを書いて実行するだけです。

**カラムナ（列指向）ストレージ**
BigQueryは内部的にデータを列単位で保存しています。分析クエリで特定のカラムだけを集計する場合、必要なカラムのデータだけを読み込むため、行指向のRDBMS（MySQLなど）と比較して大幅にI/Oを削減できます。

**DuckDBとの共通点と違い**
DuckDBも列指向のOLAPエンジンという点では共通していますが、DuckDBはローカルで動作するインプロセスデータベースです。BigQueryはクラウド上で動作し、ストレージとコンピュートの分離、自動スケーリング、公開データセットへのアクセスなど、クラウドDWHならではの機能を備えています。

## リソース階層（プロジェクト / データセット / テーブル）

BigQueryのリソースは **3層構造** で管理されます。

```
プロジェクト（my-bq-project）
  └── データセット（≒ スキーマ / データベース）
        └── テーブル / ビュー
```

### MySQL/PostgreSQLとの対応

| BigQuery | MySQL | PostgreSQL | DuckDB |
|---|---|---|---|
| プロジェクト | サーバーインスタンス | サーバーインスタンス | - |
| データセット | データベース | スキーマ | スキーマ |
| テーブル | テーブル | テーブル | テーブル |

ただし、BigQueryのプロジェクトはMySQLのサーバーインスタンスとは性質が異なります。プロジェクトは **課金と権限管理の境界（コンテナ）** であり、物理的なサーバーではありません。BigQueryでは権限さえあれば、プロジェクトAのテーブルとプロジェクトBのテーブルをJOINするといったクロスプロジェクトクエリが容易にできます。これはサーバーインスタンスが物理的に分離されているMySQLとは大きく異なる点です。

### テーブルの参照方法

BigQueryではテーブルを `プロジェクトID.データセットID.テーブルID` の3部構成で参照します。

```sql
-- 完全修飾名（別プロジェクトのテーブルを参照する場合）
SELECT * FROM `bigquery-public-data.github_repos.languages`;

-- 同じプロジェクト内なら省略可能
SELECT * FROM `bq_tutorial.users`;
```

プロジェクトIDにハイフンが含まれる場合（例: `my-bq-project`）はバッククォートで囲む必要があります。

### データセットの重要な特性

データセットには作成時に **ロケーション（リージョン）** を指定します。一度作成したデータセットのロケーションは変更できません。異なるリージョンのデータセット間でJOINすることもできないため、最初のリージョン選択は重要です。

```sql
CREATE SCHEMA IF NOT EXISTS `my-bq-project.bq_tutorial`
OPTIONS (
  location = 'asia-northeast1',
  description = 'BigQuery入門検証用データセット'
);
```

BigQueryでは `CREATE SCHEMA` と `CREATE DATASET` のどちらも使えます。

## 課金体系

BigQueryの課金は **ストレージ** と **コンピュート** に分かれています。

### コンピュート（クエリ実行）

| プラン | 料金（東京リージョン） | 特徴 |
|---|---|---|
| オンデマンド | $7.50 / TiB（スキャン量課金） | デフォルト。使った分だけ支払い |
| 容量課金（Editions） | スロット単位の時間課金 | 大規模利用向け。コスト予測が可能 |

※ USリージョンでは $6.25 / TiB です。リージョンによって料金が異なるため、公式の料金ページで確認してください。

### ストレージ

BigQueryのストレージには「論理ストレージ」と「物理ストレージ」の2つの課金モデルがあります。デフォルトは論理ストレージです。

**論理ストレージ（デフォルト）— 東京リージョン:**

| 種類 | 料金 |
|---|---|
| アクティブストレージ | $0.023 / GiB・月 |
| 長期保存（90日間未変更） | $0.016 / GiB・月 |

**物理ストレージ — 東京リージョン:**

| 種類 | 料金 |
|---|---|
| アクティブストレージ | $0.052 / GiB・月 |
| 長期保存（90日間未変更） | $0.026 / GiB・月 |

物理ストレージは圧縮後のサイズで課金されるため、圧縮率が高いデータでは物理ストレージの方が安くなるケースがあります。

### 無料枠

BigQueryには **恒久的な無料枠** があります。

- **クエリ**: 1TiB / 月（オンデマンドプランのみ）
- **ストレージ**: 10GiB / 月

これはGCPの新規アカウント向け$300無料トライアルクレジットとは別物です。無料トライアル終了後も無料枠は継続します。個人の学習・検証レベルであれば、基本的に無料枠内で十分です。

### 最低課金バイト数に注意

今回の検証で気づいた重要なポイントがあります。実際にクエリを実行して「ジョブ情報」を確認したところ、以下のような結果でした。

| クエリ | 処理されたバイト数 | 課金されるバイト数 |
|---|---|---|
| SELECT *（usersテーブル・5行） | 343 B | 10 MB |
| JOIN（users × orders） | 616 B | 20 MB |
| GROUP BY集計 | 405 B | 20 MB |

実際に処理されたデータは数百バイトなのに、課金されるバイト数は最低10MBとなっています。BigQueryのオンデマンド課金には **最低10MB** の下限があるためです。小さいテーブルに何度もクエリを投げると、実際のデータサイズ以上に課金対象バイト数が積み上がっていきます。

とはいえ、無料枠が1TiB/月なので、10MBのクエリなら10万回実行できる計算です。検証レベルでは全く気にする必要はありません。

なお、BigQueryは列指向ストレージなので、**必要なカラムだけをSELECTすることが最大の節約になります。** `SELECT *` はテーブルの全カラムをスキャンするため、大きなテーブルで不用意に実行すると想定外のコストが発生します。特に公開データセットのような大規模テーブルでは、必ず必要なカラムだけを指定する習慣をつけましょう。

## テーブル作成とデータロード

### SQLでのテーブル作成

```sql
CREATE TABLE `my-bq-project.bq_tutorial.users` (
  user_id INT64 NOT NULL,
  name STRING,
  email STRING,
  age INT64,
  prefecture STRING,
  created_at TIMESTAMP
);
```

### MySQLとの型の違い

| MySQL | BigQuery | 備考 |
|---|---|---|
| INT / BIGINT | INT64 | 整数は64bit固定 |
| VARCHAR(255) | STRING | 長さ指定不要 |
| FLOAT / DOUBLE | FLOAT64 | 浮動小数点も64bit固定 |
| DATETIME | DATETIME | タイムゾーンなし |
| TIMESTAMP | TIMESTAMP | タイムゾーン付き（UTC） |
| BOOLEAN | BOOL | - |
| JSON | JSON | BigQueryでもJSON型をサポート |

BigQueryではMySQLのように `INT` や `TINYINT` といった細かい整数型の使い分けがなく、`INT64` に統一されています。`STRING` も長さ制限の指定が不要です。型のバリエーションが少ない分、スキーマ設計はシンプルになります。

### INSERTによるデータ投入

```sql
INSERT INTO `my-bq-project.bq_tutorial.users` (user_id, name, email, age, prefecture, created_at)
VALUES
  (1, '田中太郎', 'tanaka@example.com', 28, '東京都', '2024-01-15 10:00:00 UTC'),
  (2, '佐藤花子', 'sato@example.com', 34, '大阪府', '2024-02-20 14:30:00 UTC'),
  (3, '鈴木一郎', 'suzuki@example.com', 45, '北海道', '2024-03-10 09:15:00 UTC'),
  (4, '高橋美咲', 'takahashi@example.com', 22, '東京都', '2024-04-05 16:45:00 UTC'),
  (5, '伊藤健太', 'ito@example.com', 31, '福岡県', '2024-05-12 11:20:00 UTC');
```

BigQueryのINSERTはDML（Data Manipulation Language）扱いです。MySQLのINSERTと見た目は同じですが、内部の動作は大きく異なります。BigQueryのDML（INSERT/UPDATE/DELETE）は内部的にストレージの再書き込みが発生するため、MySQLのような行単位の頻繁な更新には向いていません。大量データの投入にはINSERTではなくバッチロード（CSV/Parquetの読み込み）が推奨されます。

なお、リアルタイム性が求められる場合は、DMLとは別に **Storage Write API**（旧称: Streaming Inserts）という仕組みがあります。これはAPIを通じてデータをリアルタイムに投入するためのもので、今回の入門記事では割愛しますが、ログの即時取り込みなど実務では重要な機能です。

### CSVファイルからのバッチロード

実務では、INSERTよりもCSVやParquetファイルからのバッチロードが主流です。BigQueryコンソールのGUIから簡単にロードできます。

**手順:**

1. エクスプローラでデータセット名の右にある「︙」→「テーブルを作成」
2. ソース: 「アップロード」を選択し、CSVファイルを指定
3. テーブル名を入力
4. スキーマ: 「自動検出」にチェック
5. 「テーブルを作成」をクリック

今回は以下のようなCSVを用意し、`orders` テーブルとしてロードしました。

```csv
order_id,user_id,product_name,price,quantity,ordered_at
1001,1,ワイヤレスイヤホン,4980,1,2024-06-01 10:30:00 UTC
1002,2,USBケーブル,890,3,2024-06-02 14:15:00 UTC
...
```

スキーマの自動検出により、`order_id` や `price` は INTEGER、`product_name` は STRING、`ordered_at` は TIMESTAMP として正しく認識されました。CSVの値がすべて整数だった `price` カラムが INTEGER（INT64）になっている点は、小数値を含むデータの場合は明示的にスキーマ定義が必要になるケースがあるので注意してください。

## 基本的なSQLクエリ

### SELECT / JOIN / GROUP BY

BigQueryのSQLは **GoogleSQL**（旧称: Standard SQL）を採用しており、標準SQLとほぼ同じ構文で書けます。

```sql
-- JOINクエリ
SELECT
  u.name,
  o.product_name,
  o.price * o.quantity AS total_amount,
  o.ordered_at
FROM `bq_tutorial.orders` o
JOIN `bq_tutorial.users` u ON o.user_id = u.user_id
ORDER BY o.ordered_at;
```

```sql
-- 集計クエリ
SELECT
  u.name,
  u.prefecture,
  COUNT(*) AS order_count,
  SUM(o.price * o.quantity) AS total_spent
FROM `bq_tutorial.orders` o
JOIN `bq_tutorial.users` u ON o.user_id = u.user_id
GROUP BY u.name, u.prefecture
ORDER BY total_spent DESC;
```

MySQLやDuckDBの経験があれば、特に違和感なく書けるはずです。

### クエリ実行時のスキャン量確認

BigQueryのコンソールでは、クエリの実行前に右上に「このクエリは ○○ を処理します」と表示されます。オンデマンド課金ではスキャン量がそのまま課金対象になるため、実行前にこの表示を確認する習慣をつけましょう。

また、実行後は「ジョブ情報」タブで以下の情報が確認できます：

- **処理されたバイト数**: 実際にスキャンされたデータサイズ
- **課金されるバイト数**: 課金計算に使われるサイズ（最低10MB）
- **期間**: クエリの実行時間
- **スロット（ミリ秒）**: 使用されたコンピュートリソース

## 公開データセット

BigQueryの大きな魅力の一つが、Googleが提供する **公開データセット** です。GitHub、Stack Overflow、Wikipedia、米国国勢調査、気象データなど、多種多様なデータセットが無料で利用できます。

### 実際に試してみる

GitHubの公開データセットを使って、プログラミング言語別のリポジトリ数を集計してみました。

```sql
SELECT
  language.name AS language_name,
  COUNT(*) AS repo_count
FROM `bigquery-public-data.github_repos.languages`,
UNNEST(language) AS language
GROUP BY language_name
ORDER BY repo_count DESC
LIMIT 15;
```

**実行結果:**

| 順位 | 言語 | リポジトリ数 |
|---|---|---|
| 1 | JavaScript | 1,099,966 |
| 2 | CSS | 807,826 |
| 3 | HTML | 777,433 |
| 4 | Shell | 640,886 |
| 5 | Python | 550,905 |
| 6 | Ruby | 374,276 |

**ジョブ情報:**
- 処理されたバイト数: 56.56 MB
- 課金されるバイト数: 57 MB
- 期間: 1秒

数百万行のデータを1秒で集計できる。これがBigQueryの威力です。

### UNNEST — BigQuery特有の構文

上記クエリで使った `UNNEST` は、BigQueryで頻出する重要な構文です。配列型（ARRAY）やネスト構造（STRUCT）を展開してフラットなテーブルとして扱えます。

MySQLには配列型が存在しないため、この概念は馴染みが薄いかもしれません。具体例で説明します。

**BigQueryでの配列型のイメージ**

GitHubの `languages` テーブルの定義を `INFORMATION_SCHEMA` で確認すると、以下のようになっています。

| column_name | data_type |
|---|---|
| repo_name | STRING |
| language | ARRAY\<STRUCT\<name STRING, bytes INT64\>\> |

`language` カラムは「nameとbytesという2つのフィールドを持つ構造体（STRUCT）の配列（ARRAY）」です。実際のデータは以下のように格納されています。

```
repo_name: "facebook/react"
language: [
  { name: "C",            bytes: 5227 },
  { name: "C++",          bytes: 44290 },
  { name: "CSS",          bytes: 64972 },
  { name: "CoffeeScript", bytes: 17390 },
  { name: "HTML",         bytes: 120058 },
  { name: "JavaScript",   bytes: 6256474 },
  ...
]
```

1つのリポジトリに対して、複数の言語が **1行の中にネストされた配列** として格納されています。

**MySQLでの設計との比較**

MySQLであれば、これは `repos` テーブルと `repo_languages` テーブルに正規化して保存するのが一般的です。

```sql
-- MySQLでの正規化された設計
SELECT r.repo_name, rl.language_name
FROM repos r
JOIN repo_languages rl ON r.repo_id = rl.repo_id;
```

BigQueryでは、この「1対多」の関係を正規化せずにネスト構造のまま1行に保存できます。そしてクエリ時に `UNNEST` で展開します。

```sql
-- BigQueryでのUNNEST
SELECT
  repo_name,
  language.name AS language_name,
  language.bytes AS bytes
FROM `bigquery-public-data.github_repos.languages`,
UNNEST(language) AS language;
```

`UNNEST` の動作を図で表すと以下のようになります：

```
【UNNEST前】1行（配列がネストされた状態）
repo_name: "facebook/react"
language: [
  { name: "C",          bytes: 5227 },
  { name: "JavaScript", bytes: 6256474 },
  { name: "HTML",       bytes: 120058 }
]

    ↓ UNNEST(language) AS language

【UNNEST後】3行に展開（language.name, language.bytes でアクセス可能）
repo_name             | language.name | language.bytes
─────────────────────-|───────────────|───────────────
facebook/react        | C             | 5227
facebook/react        | JavaScript    | 6256474
facebook/react        | HTML          | 120058
```

つまり、MySQLでの「JOINで別テーブルを結合する」操作を、BigQueryでは「UNNESTで同じ行内の配列を展開する」ことで実現しています。

**ARRAY\<STRUCT\> と MySQL JSON型の比較**

MySQLでも JSON型を使えば似たような構造を表現できます。

```sql
-- MySQLでJSON型を使った場合
CREATE TABLE repo_languages (
  repo_name VARCHAR(255),
  language JSON
  -- [{"name": "JavaScript", "bytes": 6256474}, {"name": "HTML", "bytes": 120058}]
);
```

```sql
-- BigQueryの場合
CREATE TABLE repo_languages (
  repo_name STRING,
  language ARRAY<STRUCT<name STRING, bytes INT64>>
);
```

しかし、両者には重要な違いがあります。

| 観点 | MySQL JSON | BigQuery ARRAY\<STRUCT\> |
|---|---|---|
| スキーマ | なし（何でも入る） | あり（型が固定） |
| バリデーション | 実行時 | テーブル定義時 |
| クエリ構文 | `JSON_EXTRACT(col, '$.name')` | `language.name`（ドットアクセス） |
| 配列の展開 | `JSON_TABLE()`（MySQL 8.0+） | `UNNEST()` |
| パフォーマンス | インデックスなしでは遅い | 列指向で最適化済み |

BigQueryの `ARRAY<STRUCT>` はスキーマで型が保証されている分、クエリもシンプルに書けて、列指向ストレージの恩恵を受けてパフォーマンスも最適化されます。MySQLのJSON型は柔軟ですが、BigQueryのようなスキーマ付きのネスト型とは設計思想が異なります。

**なぜBigQueryではネスト構造を使うのか？**

BigQueryはスキャン量が課金に直結するため、JOINを減らすことがコスト最適化につながります。あらかじめ関連データをネスト構造で1つのテーブルにまとめておけば、JOINなしで分析できます。データの重複も発生しないため、ストレージ効率も良くなります。

### 公開データセットの課金について

公開データセットのストレージ費用はGoogleが負担しているため、ストレージ料金はかかりません。ただし、**クエリのコンピュート費用は自分のプロジェクトの無料枠から消費されます。** 大きなテーブルに対するフルスキャンを繰り返すと無料枠を使い切る可能性があるので、WHERE句やLIMITで絞り込むことを意識しましょう。

また、公開データセットのロケーションは **US** であることがジョブ情報から確認できました。自分のデータセット（asia-northeast1）とはリージョンが異なるため、公開データセットと自分のテーブルをJOINする場合はリージョンの不一致に注意が必要です。

## パーティショニングとクラスタリング

BigQueryでコスト最適化とパフォーマンス向上を実現するための最も重要な機能です。

### パーティショニングとは

テーブルを特定のカラム（通常は日付）の値に基づいて **物理的に分割** する機能です。クエリ時にWHERE句で日付を指定すると、該当するパーティションのデータだけを読み込みます（**パーティションプルーニング**）。

### クラスタリングとは

パーティション内のデータを、指定したカラムの値で **ソート・グループ化して配置** する機能です。クエリ時にクラスタリングキーで絞り込むと、関連するデータブロックだけを効率的に読み込めます。

### 実際に作成してみる

```sql
CREATE TABLE `my-bq-project.bq_tutorial.access_logs` (
  log_id INT64,
  user_id INT64,
  page_path STRING,
  action STRING,
  device_type STRING,
  created_at TIMESTAMP
)
PARTITION BY DATE(created_at)
CLUSTER BY device_type, action;
```

- `PARTITION BY DATE(created_at)`: `created_at` の日付部分でパーティション分割
- `CLUSTER BY device_type, action`: パーティション内を `device_type`、`action` の順でソート

### パーティションプルーニングの効果を確認

テストデータ（3日分・10行）を投入した後、特定の日付だけを指定してクエリを実行しました。

```sql
SELECT *
FROM `my-bq-project.bq_tutorial.access_logs`
WHERE DATE(created_at) = '2024-06-01';
```

**結果:** 4行のみ返却（2024-06-01のデータだけ）

**ジョブ情報:**
- 処理されたバイト数: **190 B**

テーブル全体ではなく、2024-06-01のパーティションだけがスキャンされていることが確認できます。

また、結果の行を見ると `device_type` でソートされていました（desktop → mobile → tablet）。これはクラスタリングの効果です。

### 実務でのインパクト

今回のテストデータは10行と小さいため効果が見えにくいですが、実務では以下のようなインパクトがあります。

**例: 1年分のログデータ（10億行）を持つテーブル**

| 条件 | パーティションなし | パーティションあり |
|---|---|---|
| 1日分のデータを取得 | 全行スキャン | 1/365だけスキャン |
| スキャン量（仮に100GB） | 100GB | 約274MB |
| オンデマンドコスト（東京） | 約$0.73 | 約$0.002 |

パーティションを設定するだけで、コストが **約360分の1** になります。

### パーティション数の上限

実務で注意すべき点として、1テーブルあたりのパーティション数は **最大10,000** という制限があります。日単位パーティションの場合は約27年分に相当するので通常は問題になりませんが、時間単位パーティションを使う場合や、数十年分のヒストリカルデータを扱う場合は上限に注意が必要です。

### DuckDBとの違い

DuckDBはローカルで動作するため、全データがメモリまたはローカルディスクに載ります。スキャン量が課金に直結するBigQueryとは異なり、パーティショニングの必要性は低いです。ただし、DuckDBでもHive形式のパーティショニングされたParquetファイルを読み込む際に、パーティションプルーニングの恩恵を受けることができます。

## DuckDBとBigQueryの比較まとめ

| 観点 | DuckDB | BigQuery |
|---|---|---|
| 実行環境 | ローカル（インプロセス） | クラウド（サーバーレス） |
| ストレージ | ローカルファイル | Google Cloud Storage |
| コスト | 完全無料 | スキャン量課金（無料枠あり） |
| スケーラビリティ | マシンスペック依存 | 自動スケーリング |
| ACID特性 | 単一ファイル内での完全なACID | テーブルレベルでのACID（DMLに制限あり） |
| データの共有 | ファイル共有が必要 | 権限設定で即共有 |
| 外部データ参照 | ローカルのParquet/CSV/JSONを直接SQLで操作 | GCS上のファイルやGoogleスプレッドシートを外部テーブルとして参照可能 |
| 公開データセット | なし | 豊富（GitHub、気象等） |
| 型システム | より柔軟（FLOAT, DOUBLEなど） | シンプル（INT64, FLOAT64に統一） |
| 配列・ネスト型 | LISTで対応 | ARRAY/STRUCTが充実、UNNESTが強力 |
| パーティショニング | Hive形式のParquetで対応可 | ネイティブサポート |
| エコシステム | Python/Rなどのライブラリとして動作 | Looker、Vertex AI等とシームレス連携 |
| 適するユースケース | ローカル分析、ETL前処理、プロトタイプ | 大規模分析基盤、データウェアハウス |

DuckDBとBigQueryは「競合」ではなく「補完関係」にあります。ローカルでの開発・プロトタイプにはDuckDBを使い、本番の大規模分析基盤にはBigQueryを使う、という使い分けが現実的です。

## まとめ

本記事ではBigQueryの基本を一通り検証しました。

**やったこと：**
- GCPプロジェクトでBigQueryコンソールを開く
- データセット・テーブルの作成（SQL & GUIからのCSVロード）
- 基本的なSQLクエリ（SELECT / JOIN / GROUP BY）
- 公開データセットへのクエリ（GitHub言語データ）
- パーティショニング・クラスタリングの作成と効果確認

**わかったこと：**
- MySQLの経験があれば、BigQueryのSQLはほぼ違和感なく書ける
- 型は `INT64` / `STRING` / `TIMESTAMP` などシンプルに統一されている
- 課金は「スキャン量」が基準。パーティショニングによるコスト最適化は必須知識
- 無料枠（1TiB/月クエリ + 10GiB/月ストレージ）は個人検証に十分
- 公開データセットは学習にもデータ分析の実務にも非常に有用
- DuckDBとは補完関係。ローカルのDuckDBで試してBigQueryに展開、という流れが自然

## 参考リンク

- [BigQuery公式ドキュメント](https://cloud.google.com/bigquery/docs)
- [BigQuery料金](https://cloud.google.com/bigquery/pricing)
- [BigQuery公開データセット](https://cloud.google.com/bigquery/public-data)
- [BigQuery SQLリファレンス](https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax)
