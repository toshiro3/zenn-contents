---
title: "MySQLエンジニアがPostgreSQLを触ってみた【データ型編】— JSONB・配列型・範囲型を比較検証"
emoji: "🐘"
type: "tech"
topics: ["PostgreSQL", "MySQL", "Database", "SQL", "Docker"]
published: false
---

## はじめに

MySQLとPostgreSQLは、どちらもよく使われるRDBMSですが、データ型やJSON周りの機能には大きな違いがあります。本記事では**同じデータ・同じクエリで挙動を比較検証**し、MySQLエンジニアの視点からPostgreSQLのデータ型の特徴を掘り下げます。

- **テーマ1**: JSONB型 vs JSON型 — 演算子・インデックス・パフォーマンスの違い
- **テーマ2**: 配列型・範囲型 — MySQLにはないPostgreSQL固有のデータ型

後編の「クエリ・運用編」（MATERIALIZED VIEW / LATERAL JOIN / EXPLAIN ANALYZE）は別記事で公開予定です。

### 検証環境

Docker ComposeでMySQL・PostgreSQLを同時に立て、同じデータ・同じクエリで挙動を比較しました。

| | バージョン |
|---|---|
| MySQL | 8.4（LTS最新） |
| PostgreSQL | 18（2025年9月GA） |
| Docker Compose | V2 |

```yaml
services:
  mysql:
    image: mysql:8.4
    container_name: compare-mysql
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: compare_db
    ports:
      - "3306:3306"
    volumes:
      - ./init/mysql:/docker-entrypoint-initdb.d
      - mysql_data:/var/lib/mysql
    command: >
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_unicode_ci

  postgres:
    image: postgres:18
    container_name: compare-postgres
    environment:
      POSTGRES_PASSWORD: rootpass
      POSTGRES_DB: compare_db
    ports:
      - "5432:5432"
    volumes:
      - ./init/postgres:/docker-entrypoint-initdb.d
      - postgres_data:/var/lib/postgresql

volumes:
  mysql_data:
  postgres_data:
```

:::message
PostgreSQL 18のDockerイメージでは、データディレクトリの構成が変更されています。ボリュームマウント先を`/var/lib/postgresql/data`ではなく`/var/lib/postgresql`にする必要があります。詳細は[docker-library/postgres#1259](https://github.com/docker-library/postgres/pull/1259)を参照してください。
:::

### テストデータ

製品データ（products）を両DBに10万件投入して検証しています。

```sql
-- 構造（共通）
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    category VARCHAR(100) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    attributes JSON,  -- PostgreSQL側はJSONB型
    tags JSON,         -- PostgreSQL側はJSONB型
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

`attributes`カラムにはブランド名、CPU、RAM、ストレージなどをJSONで格納し、`tags`にはタグの配列を格納しています。

## テーマ1: JSONB型 vs JSON型

### 格納方式の違い

MySQLのJSON型とPostgreSQLのJSONB型は、どちらも内部的にはバイナリフォーマットで保存しています。

```sql
-- 両方で実行
SELECT attributes FROM products WHERE id = 1;
```

```
-- MySQL
{"cpu": "M3 Pro", "brand": "Apple", "color": "Space Black", "ports": ["HDMI", "USB-C", "MagSafe"], "ram_gb": 18, "storage_gb": 512}

-- PostgreSQL
{"cpu": "M3 Pro", "brand": "Apple", "color": "Space Black", "ports": ["HDMI", "USB-C", "MagSafe"], "ram_gb": 18, "storage_gb": 512}
```

キーの順序は両方とも同じ結果になりました。「MySQLは挿入時のキー順序を保持する」という情報を見かけることがありますが、MySQL 8.0以降のJSON型はバイナリフォーマットで保存する際にキーをソート（キー長→辞書順）しています。PostgreSQLのJSONBも同様のソートを行うため、結果が一致します。

:::message
PostgreSQLには`JSON型`と`JSONB型`の2種類があります。JSON型はテキストそのまま保存するため挿入時のキー順序を保持しますが、検索やインデックスの面でJSONBが圧倒的に優れているため、基本的にJSONBを使います。
:::

### 演算子の違い — パス指定の構文

JSONから値を抽出する`->`（JSON値）と`->>`（テキスト値）の意味は同じですが、パスの指定方法が異なります。

```sql
-- MySQL: JSONPathの $.記法
SELECT
    name,
    attributes->'$.brand' AS brand_json,
    attributes->>'$.brand' AS brand_text,
    attributes->'$.ram_gb' AS ram_gb
FROM products
WHERE category = 'laptop';

-- PostgreSQL: キー名を直接指定
SELECT
    name,
    attributes->'brand' AS brand_json,
    attributes->>'brand' AS brand_text,
    (attributes->>'ram_gb')::int AS ram_gb
FROM products
WHERE category = 'laptop';
```

```
-- 両方とも同じ結果
        name        | brand_json | brand_text | ram_gb
--------------------+------------+------------+--------
 MacBook Pro 14"    | "Apple"    | Apple      |     18
 ThinkPad X1 Carbon | "Lenovo"   | Lenovo     |     32
 Dell XPS 15        | "Dell"     | Dell       |     32
```

MySQLは`$.brand`のようにJSONPath記法で指定し、PostgreSQLはキー名を直接渡します。PostgreSQLで数値として扱いたい場合は`::int`のようにキャストが必要です。

### 配列要素・ネストへのアクセス

JSON内の配列要素やネストした値へのアクセスで、構文の差が大きくなります。

```sql
-- MySQL: JSON_EXTRACT関数でJSONPath指定
SELECT
    name,
    JSON_EXTRACT(attributes, '$.ports[0]') AS first_port,
    JSON_EXTRACT(attributes, '$.ports') AS all_ports,
    JSON_LENGTH(JSON_EXTRACT(attributes, '$.ports')) AS port_count
FROM products
WHERE JSON_CONTAINS_PATH(attributes, 'one', '$.ports');

-- PostgreSQL: 演算子チェーンでアクセス
SELECT
    name,
    attributes->'ports'->0 AS first_port,
    attributes->'ports' AS all_ports,
    jsonb_array_length(attributes->'ports') AS port_count
FROM products
WHERE attributes ? 'ports';
```

```
-- 両方とも同じ結果
        name        | first_port |          all_ports           | port_count
--------------------+------------+------------------------------+------------
 MacBook Pro 14"    | "HDMI"     | ["HDMI", "USB-C", "MagSafe"] |          3
 ThinkPad X1 Carbon | "USB-C"    | ["USB-C", "USB-A", "HDMI"]   |          3
 Dell XPS 15        | "USB-C"    | ["USB-C", "SD Card"]         |          2
```

MySQLでは`JSON_EXTRACT`関数にJSONPathを渡す形式で、キーの存在確認も`JSON_CONTAINS_PATH`関数を使います。一方PostgreSQLでは、`->`演算子をチェーンして`attributes->'ports'->0`のようにアクセスでき、キーの存在確認も`?`演算子1文字で書けます。

### 条件検索 — PostgreSQL固有の包含演算子

JSON内の値で検索する場面で、最も大きな差が出ます。

```sql
-- MySQL: JSON_CONTAINS関数（値をJSON形式の文字列で渡す必要あり）
SELECT name FROM products
WHERE JSON_CONTAINS(attributes, '"Apple"', '$.brand');

SELECT name FROM products
WHERE JSON_CONTAINS(tags, '"flagship"');

-- PostgreSQL: @> 包含演算子
SELECT name FROM products
WHERE attributes @> '{"brand": "Apple"}';

SELECT name FROM products
WHERE tags @> '["flagship"]';
```

```
-- brand = "Apple" の結果（両方同じ）
MacBook Pro 14"
iPhone 15 Pro
iPad Air
AirPods Pro 2

-- tags に "flagship" を含む結果（両方同じ）
Galaxy S24 Ultra
iPhone 15 Pro
```

PostgreSQLの`@>`（包含演算子）は「左辺が右辺の値を含むか」を判定します。MySQLの`JSON_CONTAINS`と結果は同じですが、`@>`はGINインデックスと組み合わせることで大幅に高速化できる点が重要です（次のセクションで検証します）。

さらにPostgreSQLには`?`演算子（キーの存在確認）もあります。

```sql
-- PostgreSQLのみ: 特定キーが存在する行を検索
SELECT name FROM products
WHERE attributes ? 'features';
```

```
Galaxy S24 Ultra
iPhone 15 Pro
iPad Air
Pixel 8 Pro
AirPods Pro 2
Sony WH-1000XM5
Nintendo Switch OLED
```

MySQLで同じことをやるには`JSON_CONTAINS_PATH(attributes, 'one', '$.features')`と書く必要があります。

### インデックス戦略の違い

10万件のデータでJSON検索のパフォーマンスを比較しました。

**インデックスなし（フルスキャン）**

```sql
-- MySQL
SELECT COUNT(*) FROM products WHERE attributes->>'$.brand' = 'Apple';
-- 0.08 sec

-- PostgreSQL
SELECT COUNT(*) FROM products WHERE attributes->>'brand' = 'Apple';
-- 53 ms
```

10万件程度ではどちらも実用的な速度です。次にインデックスを作成します。

**MySQLのアプローチ — 生成カラム + B-treeインデックス**

MySQLではJSON内の値に直接インデックスを作成できません。代わりに、仮想生成カラム（Generated Column）を追加し、そこにB-treeインデックスを貼ります。

```sql
-- MySQL: 生成カラムを追加してインデックス作成
ALTER TABLE products
    ADD COLUMN attr_brand VARCHAR(100)
        GENERATED ALWAYS AS (attributes->>'$.brand') STORED,
    ADD INDEX idx_attr_brand (attr_brand);

-- 生成カラムを使って検索
SELECT COUNT(*) FROM products WHERE attr_brand = 'Apple';
-- 0.01 sec
```

```sql
EXPLAIN ANALYZE
SELECT COUNT(*) FROM products WHERE attr_brand = 'Apple';
```

```
-> Aggregate: count(0)  (cost=3961 rows=1) (actual time=4.82..4.82 rows=1 loops=1)
    -> Covering index lookup on products using idx_attr_brand (attr_brand='Apple')
       (cost=2097 rows=18640) (actual time=0.34..4.09 rows=10056 loops=1)
```

Covering index lookupが使われ、約5msで完了しています。

**PostgreSQLのアプローチ — GINインデックス**

PostgreSQLのJSONBには、GIN（Generalized Inverted Index）インデックスを直接作成できます。

```sql
-- PostgreSQL: JSONB全体にGINインデックス作成
CREATE INDEX idx_attributes_gin ON products USING GIN (attributes);

-- @> 包含演算子で検索（GINインデックスが効く）
SELECT COUNT(*) FROM products WHERE attributes @> '{"brand": "Apple"}';
-- 35 ms
```

```sql
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
  Planning Time: 0.927 ms
  Execution Time: 30.947 ms
```

Bitmap Index Scanが使われ、GINインデックスが機能しています。

**PostgreSQLでもB-treeは使える — 式インデックス**

「GINは汎用的だけど遅いのでは？」と思うかもしれませんが、PostgreSQLでも特定キーに対してB-treeインデックスを作成できます。MySQLの生成カラムに相当するのが**式インデックス（Expression Index）**です。

```sql
-- PostgreSQL: 式インデックス（カラム追加不要でB-treeを作成）
CREATE INDEX idx_attr_brand_btree ON products ((attributes->>'brand'));

-- そのまま ->> 演算子で検索すればインデックスが効く
SELECT COUNT(*) FROM products WHERE attributes->>'brand' = 'Apple';
```

```sql
EXPLAIN ANALYZE
SELECT COUNT(*) FROM products WHERE attributes->>'brand' = 'Apple';
```

```
Aggregate  (cost=1254.71..1254.72 rows=1 width=8)
           (actual time=25.334..25.341 rows=1 loops=1)
  Buffers: shared hit=2269
  ->  Bitmap Heap Scan on products  (cost=8.17..1253.46 rows=500 width=0)
                                    (actual time=1.533..23.794 rows=5555 loops=1)
        Recheck Cond: ((attributes ->> 'brand'::text) = 'Apple'::text)
        Heap Blocks: exact=2263
        Buffers: shared hit=2269
        ->  Bitmap Index Scan on idx_attr_brand_btree  (cost=0.00..8.04 rows=500 width=0)
                                                       (actual time=0.879..0.880 rows=5555 loops=1)
              Index Cond: ((attributes ->> 'brand'::text) = 'Apple'::text)
              Index Searches: 1
              Buffers: shared hit=6
  Execution Time: 25.530 ms
```

GINインデックスの31msから25.5msに改善し、インデックス部分のバッファアクセスも34→6ブロックに減っています。MySQLの生成カラム方式との大きな違いは、**テーブルにカラムを追加せずにインデックスだけ作成できる**点です。

**3方式のトレードオフ比較**

| | MySQL（生成カラム + B-tree） | PostgreSQL（GIN） | PostgreSQL（式インデックス） |
|---|---|---|---|
| 実行時間 | **約5ms** | 約31ms | 約26ms |
| インデックス対象 | 特定の1フィールド | JSONB内の全キー | 特定の1フィールド |
| カラム追加 | 必要 | 不要 | **不要** |
| brand以外の検索 | カラム+INDEX追加が必要 | そのまま使える | INDEX追加が必要 |
| ストレージ | 生成カラム+INDEX分 | GIN INDEX分 | INDEX分のみ |

単一フィールドの検索速度ではMySQLの生成カラム方式が最速です。ただしPostgreSQLでも式インデックスを使えば近い速度が出せます。一方、GINインデックスは1つ作るだけでJSON内の**全キーに対して包含検索が効く**汎用性があり、`@> '{"ram_gb": 32}'`も`@> '{"brand": "Apple", "storage_gb": 256}'`も追加インデックスなしで高速化されます。

使い分けの目安としては、検索対象のフィールドが決まっていて速度重視なら式インデックス（B-tree）、JSONの柔軟な検索が必要ならGINインデックス、という組み合わせになります。もちろん両方を併用することもできます。

## テーマ2: 配列型・範囲型 — MySQL非対応のデータ型

テーマ2では、PostgreSQLにあってMySQLにはないデータ型を検証します。MySQLで同等のことを実現しようとしたときの代替手段も比較します。

### 配列型（TEXT[]）

PostgreSQLには配列型があり、1つのカラムに複数の値をネイティブに格納できます。

**テーブル定義の違い**

```sql
-- PostgreSQL: 配列型を使用
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    tag_list TEXT[],   -- 配列型
    ...
);

INSERT INTO events (name, tag_list) VALUES
('Tech Conference 2024', ARRAY['tech', 'conference', 'networking']);

-- MySQL: JSON型で代替
CREATE TABLE events (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    tag_list JSON,     -- 配列型がないのでJSONで代替
    ...
);

INSERT INTO events (name, tag_list) VALUES
('Tech Conference 2024', '["tech", "conference", "networking"]');
```

**データの見え方**

```sql
SELECT name, tag_list FROM events;
```

```
-- PostgreSQL: 配列リテラルで表示
          name           |           tag_list
-------------------------+------------------------------
 Tech Conference 2024    | {tech,conference,networking}
 Music Festival          | {music,outdoor,festival}
 AI Workshop             | {ai,tech,hands-on}
 Data Engineering Meetup | {data,tech,meetup}
 Startup Pitch Night     | {startup,networking,pitch}
```

PostgreSQLでは`{tech,conference,networking}`のように配列リテラルで表示されます。

**配列の操作比較**

```sql
-- PostgreSQL: "tech"タグを含むイベント
SELECT name FROM events WHERE 'tech' = ANY(tag_list);

-- MySQL: JSON_CONTAINS で代替
SELECT name FROM events WHERE JSON_CONTAINS(tag_list, '"tech"');
```

```
-- 両方とも同じ結果
Tech Conference 2024
AI Workshop
Data Engineering Meetup
```

ここまでは「書き方が違うだけ」に見えますが、PostgreSQLの配列型が真価を発揮するのは**重なり判定**と**展開**です。

```sql
-- PostgreSQL: 複数値のいずれかを含むイベント（&&演算子）
SELECT name FROM events WHERE tag_list && ARRAY['music', 'startup'];
```

```
Music Festival
Startup Pitch Night
```

`&&`演算子で「配列同士に共通要素があるか」を1行で判定できます。MySQLのJSON型で同じことをやるには`JSON_CONTAINS(tag_list, '"music"') OR JSON_CONTAINS(tag_list, '"startup"')`とOR条件を列挙する必要があります。

```sql
-- PostgreSQL: 配列を行に展開（unnest）
SELECT name, unnest(tag_list) AS tag
FROM events WHERE name = 'Tech Conference 2024';
```

```
         name         |    tag
----------------------+------------
 Tech Conference 2024 | tech
 Tech Conference 2024 | conference
 Tech Conference 2024 | networking
```

`unnest`で配列を1要素1行に展開できます。タグごとの集計やJOINに便利です。MySQLでは`JSON_TABLE`関数で同様の展開が可能ですが、構文がかなり重くなります。

**配列型の操作まとめ**

| 操作 | PostgreSQL | MySQL（JSON代替） |
|---|---|---|
| 値の包含 | `'tech' = ANY(tag_list)` | `JSON_CONTAINS(tag_list, '"tech"')` |
| 複数値の重なり | `tag_list && ARRAY['music', 'startup']` | OR条件を列挙 |
| 要素数 | `array_length(tag_list, 1)` | `JSON_LENGTH(tag_list)` |
| 行への展開 | `unnest(tag_list)` | `JSON_TABLE`（構文が重い） |

### 範囲型（TSRANGE / INT4RANGE）

PostgreSQLの範囲型は、「開始値と終了値のペア」を1つの型として扱えます。MySQLにはこの型が存在しないため、2つのカラムに分割して管理する必要があります。

**テーブル定義の違い**

```sql
-- PostgreSQL: 範囲型を使用
CREATE TABLE events (
    ...
    schedule TSRANGE,        -- タイムスタンプの範囲
    price_range INT4RANGE,   -- 整数の範囲
);

INSERT INTO events (name, schedule, price_range) VALUES
('Tech Conference 2024',
 '[2024-09-01 09:00, 2024-09-03 18:00)',  -- [ は下限含む、) は上限含まない
 '[5000, 15000)');

-- MySQL: 2カラムに分割
CREATE TABLE events (
    ...
    schedule_start DATETIME,
    schedule_end DATETIME,
    price_min INT,
    price_max INT,
);
```

PostgreSQLの範囲型は`[)`（下限含む・上限含まない＝半開区間）を型レベルで表現できます。この「半開区間」は実務では非常に重要で、例えばイベントの終了時刻が18:00のとき「18:00ちょうどは含まない」ことを明示できます。

**範囲内の判定**

```sql
-- PostgreSQL: @> 演算子で「範囲内か」を判定
SELECT name FROM events
WHERE schedule @> '2024-09-02 12:00'::timestamp;

-- MySQL: BETWEENで代替
SELECT name FROM events
WHERE '2024-09-02 12:00:00' BETWEEN schedule_start AND schedule_end;
```

```
-- 両方とも同じ結果
Tech Conference 2024
```

1対1の判定では大きな差はありませんが、MySQLの`BETWEEN`は`<=`（以下）なので、`schedule_end`ちょうどの値を含むかどうかの挙動がPostgreSQLの`[)`とは異なります。

**範囲同士の重なり判定**

ここが範囲型の最大の強みです。

```sql
-- PostgreSQL: && 演算子で「範囲が重なるか」を判定
SELECT name FROM events
WHERE schedule && '[2024-10-01 00:00, 2024-10-31 23:59)'::tsrange;

-- 予算5,000〜10,000円と重なるイベント
SELECT name, price_range FROM events
WHERE price_range && '[5000, 10000)'::int4range;

-- MySQL: 手動で不等式を記述
SELECT name FROM events
WHERE schedule_start < '2024-10-31 23:59:00'
  AND schedule_end > '2024-10-01 00:00:00';

SELECT name, price_min, price_max FROM events
WHERE price_min < 10000 AND price_max > 5000;
```

```
-- スケジュールの重なり（両方同じ結果）
AI Workshop

-- 価格帯の重なり（両方同じ結果）
Tech Conference 2024
Music Festival
```

PostgreSQLでは`&&`の1演算子で済む一方、MySQLでは`start < 上限 AND end > 下限`という不等式を手動で記述する必要があります。この手動での記述は**境界条件のバグ（`<` vs `<=`）が入り込みやすい**のが実務上の課題です。

**境界値の取得と計算**

```sql
-- PostgreSQL: lower() / upper() で境界値を取得、差分も計算可能
SELECT name,
    lower(schedule) AS starts_at,
    upper(schedule) AS ends_at,
    upper(schedule) - lower(schedule) AS duration
FROM events;
```

```
          name           |      starts_at      |       ends_at       |    duration
-------------------------+---------------------+---------------------+-----------------
 Tech Conference 2024    | 2024-09-01 09:00:00 | 2024-09-03 18:00:00 | 2 days 09:00:00
 Music Festival          | 2024-07-20 12:00:00 | 2024-07-22 23:00:00 | 2 days 11:00:00
 AI Workshop             | 2024-10-15 10:00:00 | 2024-10-15 17:00:00 | 07:00:00
 Data Engineering Meetup | 2024-08-10 19:00:00 | 2024-08-10 21:00:00 | 02:00:00
 Startup Pitch Night     | 2024-11-01 18:00:00 | 2024-11-01 22:00:00 | 04:00:00
```

`lower()`/`upper()`で境界値を取得でき、差分計算もTimestampの引き算として自然に書けます。「イベントの期間が最も長いものを検索」といったクエリも簡潔に書けます。

**範囲型の操作まとめ**

| 操作 | PostgreSQL | MySQL |
|---|---|---|
| 範囲の表現 | `TSRANGE`型1カラム | `DATETIME`型2カラム |
| 半開区間 `[)` | 型レベルで表現 | アプリ側で制御が必要 |
| 値が範囲内か | `schedule @> timestamp` | `BETWEEN start AND end` |
| 範囲の重なり | `range1 && range2` | `start < 上限 AND end > 下限` |
| 境界値の取得 | `lower()` / `upper()` | そのままカラム参照 |
| GiSTインデックス | 対応 | — |

### 範囲型が活きるユースケース

範囲型は以下のような場面で特に威力を発揮します。

- **予約システム** — 会議室の予約が重複しないか`&&`で判定。排他制約（EXCLUDE制約）と組み合わせるとDB側でダブルブッキングを防止できる
- **料金体系** — 日付範囲ごとの料金設定で、有効期間の重なりチェック
- **バージョン管理** — 有効期間付きのマスターデータで、特定時点の値を取得

MySQLでも同じことは実現できますが、アプリケーション側でのバリデーションロジックが増えます。PostgreSQLでは型とDB制約に任せられるため、バグの入り込む余地が減ります。

## まとめ

| 機能 | MySQL 8.4 | PostgreSQL 18 |
|---|---|---|
| JSON型 | JSON型（バイナリ格納） | JSON型 + **JSONB型** |
| JSON演算子 | `$.`パス記法 | キー直接指定 + `@>`, `?` 等の専用演算子 |
| JSONインデックス | 生成カラム + B-tree | **GINインデックス（全キー対応）** |
| 配列型 | なし（JSON代替） | **TEXT[] 等のネイティブ配列型** |
| 範囲型 | なし（2カラム分割） | **TSRANGE, INT4RANGE 等** |

MySQLでもJSON型や複数カラムの組み合わせで同等の結果は得られます。ただし構文の簡潔さ、インデックスの汎用性、型レベルでの制約表現という点でPostgreSQLに優位性があると感じました。

特に範囲型は、予約やスケジュール管理のような「期間の重なり」を扱うシステムで大きな差になります。MySQLで`start < 上限 AND end > 下限`を手動で書くのと、PostgreSQLで`&&`の1演算子で済むのでは、コードの可読性だけでなくバグの発生率にも影響します。

次回の「クエリ・運用編」では、MATERIALIZED VIEW、LATERAL JOIN、EXPLAIN ANALYZEの違いを比較します。
