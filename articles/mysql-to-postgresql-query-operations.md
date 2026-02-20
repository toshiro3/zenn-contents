---
title: "MySQLエンジニアがPostgreSQLを触ってみた【クエリ・運用編】LATERAL JOIN・MView・EXPLAIN比較検証"
emoji: "🐘"
type: "tech"
topics: ["PostgreSQL", "MySQL", "Database", "SQL", "Docker"]
published: false
---

## はじめに

前回の「データ型編」ではJSONB・配列型・範囲型をMySQLと比較しました。

https://zenn.dev/toshiro3/articles/mysql-to-postgresql-data-types

本記事は「クエリ・運用編」として、クエリの書き方やDB運用に関わる以下のテーマを比較検証します。

- **テーマ1**: LATERAL JOIN — サブクエリ内で外側の行を参照する
- **テーマ2**: MATERIALIZED VIEW — 集計結果を実体として保存する
- **テーマ3**: EXPLAIN ANALYZE — 実行計画の情報量の違い

### 検証環境

前回と同じDocker Compose環境（MySQL 8.4 + PostgreSQL 18）を使用しています。テストデータとして、ユーザー5人と注文13件、製品10万件を投入しています。

```sql
-- テーブル構成（両DB共通）
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    username VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL
);

CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    product_id BIGINT NOT NULL REFERENCES products(id),
    quantity INT NOT NULL DEFAULT 1,
    total_price DECIMAL(10, 2) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    ordered_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## テーマ1: LATERAL JOIN

### LATERAL JOINとは

通常のJOINでは、サブクエリは外側のクエリの値を参照できません。LATERAL JOINを使うと、サブクエリ内で**外側のテーブルの各行の値を参照できる**ようになります。

```sql
-- 通常のJOIN → エラー（サブクエリ内でu.idは参照できない）
SELECT * FROM users u
JOIN (
    SELECT * FROM orders WHERE user_id = u.id
) o ON true;

-- LATERAL JOIN → OK（サブクエリ内でu.idを参照できる）
SELECT * FROM users u
JOIN LATERAL (
    SELECT * FROM orders WHERE user_id = u.id
) o ON true;
```

イメージとしては、外側のテーブルを1行ずつ処理しながら、その行の値を使ってサブクエリを実行します。プログラミングのforループに近い動きですが、DBエンジンが最適化してくれます。

### 「各ユーザーの最新注文2件」を取得する

LATERAL JOINが特に有効なのは、**「グループごとのTop N件」**を取得するケースです。

```sql
-- MySQL / PostgreSQL 共通（MySQL 8.0.14以降で対応）
SELECT u.username, o.id AS order_id, o.total_price, o.status, o.ordered_at
FROM users u
JOIN LATERAL (
    SELECT *
    FROM orders
    WHERE user_id = u.id
    ORDER BY ordered_at DESC
    LIMIT 2
) o ON true
ORDER BY u.username, o.ordered_at DESC;
```

```
 username  | order_id | total_price |  status   |     ordered_at
-----------+----------+-------------+-----------+---------------------
 sato      |       10 |   229800.00 | pending   | 2024-08-01 10:00:00
 sato      |        9 |   129900.00 | completed | 2024-03-01 12:00:00
 suzuki    |        6 |   159800.00 | shipped   | 2024-07-15 13:00:00
 suzuki    |        5 |    49800.00 | completed | 2024-03-10 16:00:00
 takahashi |       13 |    79600.00 | shipped   | 2024-07-01 09:00:00
 takahashi |       12 |   249800.00 | completed | 2024-05-20 11:30:00
 tanaka    |        3 |    98800.00 | completed | 2024-06-01 09:00:00
 tanaka    |        2 |    39800.00 | completed | 2024-02-20 14:30:00
 yamada    |        8 |    37980.00 | completed | 2024-04-05 19:00:00
 yamada    |        7 |   198000.00 | completed | 2024-02-01 08:30:00
```

ユーザーごとにサブクエリが実行され、`ORDER BY ... LIMIT 2`で最新2件だけが取得されます。MySQL 8.4・PostgreSQL 18ともに同じ構文・同じ結果です。

:::message
LATERAL JOINは「PostgreSQL固有の機能」と紹介されていることがありますが、MySQL 8.0.14（2019年）以降でサポートされています。MySQL 8.4でも問題なく使えます。
:::

### ウィンドウ関数との比較

同じ結果はROW_NUMBER()ウィンドウ関数でも得られます。

```sql
-- MySQL / PostgreSQL 共通
SELECT username, order_id, total_price, status, ordered_at
FROM (
    SELECT u.username, o.id AS order_id, o.total_price, o.status, o.ordered_at,
        ROW_NUMBER() OVER (PARTITION BY u.id ORDER BY o.ordered_at DESC) AS rn
    FROM users u
    JOIN orders o ON o.user_id = u.id
) ranked
WHERE rn <= 2
ORDER BY username, ordered_at DESC;
```

結果は同じですが、処理の流れが異なります。

| | LATERAL JOIN | ウィンドウ関数（ROW_NUMBER） |
|---|---|---|
| 処理の流れ | ユーザーごとに2件だけ取得 | 全件JOINしてから行番号で絞り込み |
| 意図の明確さ | 「各ユーザーのTop 2」が構文から読み取れる | サブクエリ + WHERE rn <= 2 の二段構え |
| データ量が多い場合 | 必要な行だけ取得するため有利 | 中間結果が膨らむ可能性 |

どちらを使うかはチームの慣れや可読性次第ですが、LATERAL JOINは「やりたいこと」がSQLの構造にそのまま反映される点が魅力です。

### LATERAL JOINが特に効く場面

「グループごとのTop N」以外にも、以下のケースでLATERAL JOINが活きます。

- **集合を返す関数との組み合わせ**: PostgreSQLの`unnest()`や`generate_series()`と組み合わせて、1行を複数行に展開
- **相関サブクエリの拡張**: 通常の相関サブクエリはスカラー値（1行1列）しか返せないが、LATERALなら複数行・複数列を返せる
- **API的なクエリ**: 「ユーザーごとに最新の通知3件とフォロワー数を一度に取得」といった、複数の関連データをまとめて取得するクエリ

なお、LATERAL JOINのサブクエリ内では`WHERE user_id = u.id ORDER BY ordered_at DESC LIMIT 2`のように外側の行で絞り込んでからソート・LIMITしています。`(user_id, ordered_at)`に複合インデックスがあれば、ユーザーごとにIndex Scanで2件だけ取得する形になるため、データ量が大きくなっても実用的な速度で動作します。

## テーマ2: MATERIALIZED VIEW

### 通常のVIEW vs MATERIALIZED VIEW

VIEWはSQLクエリに名前をつけて再利用するための機能ですが、MySQLとPostgreSQLではVIEWの種類に差があります。

**MySQLの通常VIEW**

```sql
-- MySQL: 通常のVIEW（クエリ定義のみ保存）
CREATE VIEW user_order_summary AS
SELECT
    u.id AS user_id,
    u.username,
    COUNT(o.id) AS order_count,
    SUM(o.total_price) AS total_spent,
    MAX(o.ordered_at) AS last_order_at
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.username;

SELECT * FROM user_order_summary ORDER BY total_spent DESC;
```

**PostgreSQLのMATERIALIZED VIEW**

```sql
-- PostgreSQL: MATERIALIZED VIEW（クエリ結果をテーブルとして保存）
CREATE MATERIALIZED VIEW user_order_summary AS
SELECT
    u.id AS user_id,
    u.username,
    COUNT(o.id) AS order_count,
    SUM(o.total_price) AS total_spent,
    MAX(o.ordered_at) AS last_order_at
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.username;

SELECT * FROM user_order_summary ORDER BY total_spent DESC;
```

```
-- 両方とも同じ結果
 user_id | username  | order_count | total_spent |    last_order_at
---------+-----------+-------------+-------------+---------------------
       5 | takahashi |           3 |   489200.00 | 2024-07-01 09:00:00
       2 | suzuki    |           3 |   399400.00 | 2024-07-15 13:00:00
       1 | tanaka    |           3 |   388400.00 | 2024-06-01 09:00:00
       4 | sato      |           2 |   359700.00 | 2024-08-01 10:00:00
       3 | yamada    |           2 |   235980.00 | 2024-04-05 19:00:00
```

結果は同じですが、仕組みが根本的に異なります。

### 決定的な違い

| | MySQL（通常のVIEW） | PostgreSQL（MATERIALIZED VIEW） |
|---|---|---|
| データの実体 | なし（クエリ定義のみ保存） | **テーブルとして保存される** |
| SELECT時の挙動 | **毎回JOINとGROUP BYを実行** | 保存済みデータを読むだけ |
| データの鮮度 | 常に最新 | リフレッシュするまで古いまま |
| インデックス | 貼れない | **貼れる** |
| ストレージ | 不要 | 結果分の容量が必要 |

MySQLの通常VIEWは「クエリのエイリアス」に過ぎません。`SELECT * FROM user_order_summary`を実行するたびに、裏でJOIN+GROUP BYの集計クエリが走ります。データ量が増えれば増えるほど遅くなります。

一方、PostgreSQLのMATERIALIZED VIEWは結果をテーブルとして物理的に保存しています。SELECTは保存済みデータを読むだけなので高速です。

### リフレッシュの仕組み

MATERIALIZED VIEWはデータの自動更新はされません。元テーブルのデータが変わったら、手動でリフレッシュします。

```sql
-- データ更新後にリフレッシュ
REFRESH MATERIALIZED VIEW user_order_summary;

-- 参照をブロックしないリフレッシュ（UNIQUE INDEXが必要）
CREATE UNIQUE INDEX ON user_order_summary (user_id);
REFRESH MATERIALIZED VIEW CONCURRENTLY user_order_summary;
```

`CONCURRENTLY`オプションを使うと、リフレッシュ中もSELECTがブロックされません。ダッシュボードの集計テーブルなど、多少の遅延が許容されるユースケースで有効です。

:::message alert
`REFRESH MATERIALIZED VIEW CONCURRENTLY`を使うには、MATERIALIZED VIEW側に**UNIQUE INDEXが必須**です。上記の例では`user_id`にUNIQUE INDEXを作成しています。UNIQUE INDEXがない状態で`CONCURRENTLY`を指定するとエラーになります。
:::

### MATERIALIZED VIEWにインデックスを貼る

MATERIALIZED VIEWはテーブルと同じようにインデックスを作成できます。

```sql
-- PostgreSQL
CREATE INDEX ON user_order_summary (total_spent DESC);
CREATE INDEX ON user_order_summary (username);
```

MySQLの通常VIEWではインデックスを貼れないため、集計結果に対する高速な検索はできません。

### MySQLでの代替手段

MySQLでMATERIALIZED VIEWに相当するものを実現するには、以下のアプローチがあります。

- **集計テーブル + バッチ処理**: 別テーブルを作り、定期的にINSERT ... SELECTやTRUNCATE + INSERTで更新する
- **イベント駆動での更新**: トリガーやアプリケーション側で注文追加時にサマリーテーブルを更新する

いずれもアプリケーション側での仕組みが必要です。PostgreSQLでは`CREATE MATERIALIZED VIEW` + `REFRESH`というDB組み込みの機能で完結します。

### どんな場面で使うか

MATERIALIZED VIEWは以下のようなケースで有効です。

- **ダッシュボードの集計**: 売上サマリー、KPI集計など、リアルタイム性よりも参照速度が重要
- **重い集計クエリの結果キャッシュ**: 複数テーブルのJOIN+GROUP BYの結果を事前計算
- **検索用のデノーマライズテーブル**: 正規化されたデータを検索しやすい形に変換

注意点として、MATERIALIZED VIEWはリフレッシュしないとデータが古いままです。リアルタイム性が必要な場面には向きません。リフレッシュの頻度はユースケースに応じて設計する必要があります。

## テーマ3: EXPLAIN ANALYZEの情報量の違い

パフォーマンスチューニングの際にまず見るのがEXPLAIN ANALYZEの実行計画です。MySQLとPostgreSQLで出力される情報量に大きな差があります。

### 同じクエリの実行計画を比較

```sql
-- MySQL / PostgreSQL 共通
EXPLAIN ANALYZE
SELECT p.name, p.category, p.price
FROM products p
JOIN orders o ON o.product_id = p.id
WHERE o.status = 'completed'
ORDER BY o.ordered_at DESC
LIMIT 5;
```

**MySQL 8.4の出力**

```
-> Limit: 5 row(s)  (cost=0.7 rows=1) (actual time=1.05..1.07 rows=5 loops=1)
    -> Nested loop inner join  (cost=0.7 rows=1) (actual time=1.02..1.04 rows=5 loops=1)
        -> Sort: o.ordered_at DESC  (cost=0.35 rows=1) (actual time=0.837..0.839 rows=5 loops=1)
            -> Filter: (o.`status` = 'completed')  (cost=0.35 rows=1)
                                                    (actual time=0.432..0.496 rows=10 loops=1)
                -> Table scan on o  (cost=0.35 rows=1) (actual time=0.384..0.442 rows=13 loops=1)
        -> Single-row index lookup on p using PRIMARY (id=o.product_id)
           (cost=0.35 rows=1) (actual time=0.0332..0.0333 rows=1 loops=5)
```

**PostgreSQL 18の出力**

```
Limit  (cost=9.48..9.49 rows=1 width=38)
       (actual time=0.225..0.227 rows=5.00 loops=1)
  Buffers: shared hit=31
  ->  Sort  (cost=9.48..9.49 rows=1 width=38)
            (actual time=0.224..0.225 rows=5.00 loops=1)
        Sort Key: o.ordered_at DESC
        Sort Method: quicksort  Memory: 25kB
        Buffers: shared hit=31
        ->  Nested Loop  (cost=0.29..9.47 rows=1 width=38)
                         (actual time=0.192..0.211 rows=10.00 loops=1)
              Buffers: shared hit=31
              ->  Seq Scan on orders o  (cost=0.00..1.16 rows=1 width=16)
                                        (actual time=0.079..0.083 rows=10.00 loops=1)
                    Filter: ((status)::text = 'completed'::text)
                    Rows Removed by Filter: 3
                    Buffers: shared hit=1
              ->  Index Scan using products_pkey on products p  (cost=0.29..8.31 rows=1 width=38)
                                                                (actual time=0.012..0.012 rows=1.00 loops=10)
                    Index Cond: (id = o.product_id)
                    Index Searches: 10
                    Buffers: shared hit=30
  Planning Time: 0.266 ms
  Execution Time: 0.274 ms
```

### 情報量の比較

| 情報 | MySQL 8.4 | PostgreSQL 18 |
|---|---|---|
| コスト見積もり | ✅ | ✅ |
| 実行時間 | ✅ | ✅ |
| ソートアルゴリズム | ❌ | ✅ quicksort, Memory: 25kB |
| バッファヒット | ❌ | ✅ shared hit=31 |
| フィルタ除外行数 | △ `filtered`（推定パーセンテージ） | ✅ Rows Removed by Filter: 3（実数） |
| Hash結合の詳細 | ❌ | ✅ Buckets, Batches, Memory Usage |
| Planning Time | ❌ | ✅ 0.266 ms |
| Index Searches | ❌ | ✅ 10 |

PostgreSQLの出力はノードごとにバッファ情報やフィルタの除外行数が表示されるため、**どのノードがボトルネックか**を特定しやすくなっています。

なお、MySQLでも`EXPLAIN`（`ANALYZE`なし）の出力に`filtered`という項目があり、フィルタ後に残る行の推定パーセンテージを確認できます。ただしこれはオプティマイザの推定値であり、PostgreSQLの`Rows Removed by Filter`のような**実測の行数**ではありません。実数で確認できる方がデバッグ時の精度は高くなります。

### PostgreSQL 18の変更点: BUFFERSがデフォルトで出力

PostgreSQL 17まではバッファ情報を見るために`EXPLAIN (ANALYZE, BUFFERS)`と明示する必要がありました。PostgreSQL 18からは`EXPLAIN ANALYZE`だけでバッファ情報が出力されます。

```sql
-- PostgreSQL 18: どちらも同じ出力になる
EXPLAIN ANALYZE SELECT ...;
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```

実際に両方のクエリを実行して確認したところ、出力内容は完全に一致しました。バッファ情報は「ディスクI/Oが発生しているか（shared read）」「キャッシュで済んでいるか（shared hit）」を判断する上で重要な情報なので、デフォルトで見られるようになったのは大きな改善です。

MySQLの`EXPLAIN ANALYZE`では主に「時間」でボトルネックを判断しますが、実行時間はその時点のサーバー負荷やキャッシュ状態に左右されるため、計測のたびに値がブレます。一方PostgreSQLのバッファアクセス数は、クエリの構造やデータ量が同じであれば基本的に一定です。**再現性の高いチューニング指標**として使えるため、「このクエリは改善できたのか？」を正確に判断しやすくなります。

### JSON検索でのインデックス有無の比較

EXPLAIN ANALYZEの実用例として、前回の「データ型編」で検証したJSON検索のインデックス効果を確認します。

**MySQLの実行計画**

```sql
-- インデックスなし（->>でJSON値を抽出して比較）
EXPLAIN ANALYZE
SELECT COUNT(*) FROM products WHERE attributes->>'$.brand' = 'Apple';
```

```
-> Aggregate: count(0)  (cost=4618 rows=1) (actual time=27.3..27.3 rows=1 loops=1)
    -> Index lookup on products using idx_attr_brand (attr_brand='Apple')
       (cost=2754 rows=18640) (actual time=0.108..26.5 rows=10056 loops=1)
```

`attributes->>'$.brand'`による検索でも、生成カラム`attr_brand`のインデックスが自動的に使われています。MySQLのオプティマイザが生成カラムの式と一致することを認識して、インデックスを選択しています。

```sql
-- 生成カラムを直接指定した場合
EXPLAIN ANALYZE
SELECT COUNT(*) FROM products WHERE attr_brand = 'Apple';
```

```
-> Aggregate: count(0)  (cost=3961 rows=1) (actual time=4.82..4.82 rows=1 loops=1)
    -> Covering index lookup on products using idx_attr_brand (attr_brand='Apple')
       (cost=2097 rows=18640) (actual time=0.34..4.09 rows=10056 loops=1)
```

生成カラムを直接指定するとCovering index lookupになり、テーブル本体へのアクセスが不要になるためさらに高速です（27ms → 5ms）。

**PostgreSQLの実行計画**

```sql
-- GINインデックスなし（->>でテキスト抽出して比較）
EXPLAIN ANALYZE
SELECT COUNT(*) FROM products WHERE attributes->>'brand' = 'Apple';
```

```
Aggregate  (cost=4002.40..4002.41 rows=1 width=8)
           (actual time=49.958..49.963 rows=1 loops=1)
  Buffers: shared hit=2501
  ->  Seq Scan on products  (cost=0.00..4001.15 rows=500 width=0)
                             (actual time=0.241..48.865 rows=5555 loops=1)
        Filter: ((attributes ->> 'brand'::text) = 'Apple'::text)
        Rows Removed by Filter: 94455
        Buffers: shared hit=2501
  Execution Time: 50.200 ms
```

`->>`演算子による抽出はGINインデックスの対象外なのでSeq Scan（全行スキャン）になります。`Rows Removed by Filter: 94455`から、10万件中94,455件がフィルタで除外されていることがわかります。

```sql
-- GINインデックスあり（@>包含演算子で検索）
EXPLAIN ANALYZE
SELECT COUNT(*) FROM products WHERE attributes @> '{"brand": "Apple"}';
```

```
Aggregate  (cost=2628.88..2628.89 rows=1 width=8)
           (actual time=30.718..30.725 rows=1 loops=1)
  Buffers: shared hit=2297
  ->  Bitmap Heap Scan on products  (cost=52.13..2616.25 rows=5050 width=0)
                                    (actual time=4.440..29.604 rows=5555 loops=1)
        Recheck Cond: (attributes @> '{"brand": "Apple"}'::jsonb)
        Heap Blocks: exact=2263
        Buffers: shared hit=2297
        ->  Bitmap Index Scan on idx_attributes_gin  (cost=0.00..50.87 rows=5050 width=0)
                                                     (actual time=3.512..3.513 rows=5555 loops=1)
              Index Cond: (attributes @> '{"brand": "Apple"}'::jsonb)
              Index Searches: 1
              Buffers: shared hit=34
  Execution Time: 30.947 ms
```

`@>`演算子に切り替えるとBitmap Index Scanが使われ、Seq Scanの50msから31msに改善しています。さらに各ノードのバッファ情報から、インデックス部分は`shared hit=34`（34ブロック）で済んでいるのに対し、Heap Scan（テーブル本体）で`shared hit=2263`ブロックのアクセスが発生していることも読み取れます。

:::message
PostgreSQLのGINインデックスは`@>`（包含演算子）や`?`（キー存在確認）に対して効きますが、`->>`（テキスト抽出）には効きません。GINインデックスの恩恵を受けるにはクエリの書き方を意識する必要があります。
:::

### 実行計画の読み方のコツ

PostgreSQLのEXPLAIN ANALYZEを読む際に注目すべきポイントをまとめます。

- **Seq Scan**: 全行スキャン。大きなテーブルでSeq Scanが出ていたら、インデックスの追加やクエリの書き換えを検討する
- **Buffers: shared hit vs shared read**: hitはキャッシュ済み、readはディスクからの読み取り。readが多い場合はメモリ不足の可能性
- **Rows Removed by Filter**: フィルタで除外された行数。この数が多いほど無駄なスキャンが発生している
- **Sort Method: external merge**: メモリ内でソートしきれずディスクを使った場合に表示される。`work_mem`の増加を検討
- **Planning Time vs Execution Time**: プランニングに時間がかかる場合は、テーブルの統計情報が古い可能性（`ANALYZE`の実行を検討）

## まとめ

| 機能 | MySQL 8.4 | PostgreSQL 18 |
|---|---|---|
| LATERAL JOIN | ✅ 8.0.14以降で対応 | ✅ |
| MATERIALIZED VIEW | ❌ 通常VIEWのみ | ✅ リフレッシュ+インデックスも可能 |
| EXPLAIN情報量 | コスト・実行時間 | バッファ・フィルタ除外行・ソート詳細等 |
| EXPLAIN BUFFERS | — | デフォルトで出力（18の新機能） |

LATERAL JOINは両方で使えるため、「PostgreSQL固有」と思って避けていたMySQL開発者は試してみる価値があります。

MATERIALIZED VIEWとEXPLAIN ANALYZEの情報量は、PostgreSQLの明確な優位点です。特にEXPLAIN ANALYZEのバッファ情報は、パフォーマンス問題の原因特定に直結するため、この差は実運用で大きく響きます。

一方で、MySQLにはMySQLの強みがあります。生成カラムによるCovering index lookupの高速さはPostgreSQLのGINインデックスでは到達できない速度でしたし、長年の実績による安定したレプリケーション運用や、クラウドサービス（Aurora MySQL等）のエコシステムの充実度は見逃せません。

「どちらが優れている」ではなく、ユースケースに応じて適切に選択する判断力が重要だと感じました。MySQLエンジニアとしてPostgreSQLを触ることで、普段使っているMySQLの特徴もより深く理解できるようになったのが最大の収穫です。

### 参考

- [PostgreSQL 18 Released!](https://www.postgresql.org/about/news/postgresql-18-released-3142/)
- [MySQL 8.4 Reference Manual](https://dev.mysql.com/doc/refman/8.4/en/)
- [PostgreSQL 18 Release Notes](https://www.postgresql.org/docs/release/18.0/)
