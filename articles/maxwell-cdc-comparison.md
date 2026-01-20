---
title: "MaxwellでMySQLのbinlogをキャプチャする - Debeziumと比較してみた"
emoji: "⚡"
type: "tech"
topics: ["MySQL", "CDC", "Maxwell", "Debezium", "Docker"]
published: false
---

## はじめに

MySQLのCDCツール **Maxwell** を検証します。

https://github.com/zendesk/maxwell

Maxwellは Zendesk 社が開発したオープンソースのCDCツールで、MySQLのbinlogを読み取り、変更イベントをJSON形式で出力します。

### この記事で確認すること

- Docker環境でのMaxwellセットアップ
- 出力されるJSONフォーマットの確認
- MySQLからレプリカクライアントとして認識されるか
- Debeziumとのアーキテクチャ・運用面での違い

比較対象のDebeziumについては以下の記事で検証しています。

https://zenn.dev/toshiro3/articles/debezium-cdc-introduction

### 検証用リポジトリ

本記事の検証環境は以下のリポジトリで公開しています。

https://github.com/toshiro3/maxwell-cdc-demo

---

## Debeziumとの違い（先に結論）

| 観点 | Maxwell | Debezium |
|------|---------|----------|
| 依存関係 | **単体で動作** | Kafka Connect必須 |
| セットアップ | シンプル | Kafka Connect含め複雑 |
| 出力先 | stdout/Kafka/Kinesis/等 | Kafka（経由で他へ） |
| JSON形式 | フラット・軽量 | スキーマ込み・リッチ |
| 対応DB | MySQLのみ | MySQL/PostgreSQL/MongoDB/等 |
| DDLキャプチャ | 対応 | 対応 |
| 初期スナップショット | 対応（`maxwell-bootstrap`） | 対応（初期ロード機能） |
| ユースケース | 軽量CDC、ストリーム連携 | 本格的なイベント駆動基盤 |

**Maxwellの立ち位置**: 「MySQLに特化した軽量CDC」という明確なポジショニング。Kafka Connectのエコシステムが不要な場合や、シンプルにbinlogをストリーム化したい場合に有力な選択肢です。

また、Maxwellには `maxwell-bootstrap` という別ユーティリティがあり、既存データの全件転送も可能です。「軽量ながら初期ロードもできる」点は大きなメリットです。

---

## 検証環境

### アーキテクチャ

```
┌─────────┐      ┌──────────┐      ┌─────────────┐
│  MySQL  │─────▶│ Maxwell  │─────▶│   stdout    │
│ (binlog)│      │  (単体)  │      │             │
└─────────┘      └──────────┘      └─────────────┘
                     │
                     ▼
              ┌──────────────┐
              │ maxwell DB   │
              │ (position等) │
              └──────────────┘
```

Debeziumとの構成の違いが一目瞭然です。Kafka Connectが不要で、Maxwellが直接MySQLのbinlogを購読します。

### 主要な設定ポイント

検証環境の全体は[リポジトリ](https://github.com/toshiro3/maxwell-cdc-demo)を参照してください。ここでは重要な設定のみ抜粋します。

#### MySQLのbinlog設定

```yaml
command:
  - --server-id=1
  - --log-bin=mysql-bin
  - --binlog-format=ROW        # 必須：行ベースレプリケーション
  - --binlog-row-image=FULL    # UPDATE時のold値取得に推奨
  - --gtid-mode=ON
  - --enforce-gtid-consistency=ON
```

`binlog-row-image=FULL` は、UPDATE時に変更前の全カラム値をbinlogに記録します。これにより、MaxwellのJSON出力で `old` フィールドに変更前の値を含めることができます。

#### Maxwellの設定

zendesk/maxwellイメージでは環境変数が認識されないため、コマンドライン引数で設定します。

```yaml
command: >
  sh -c "sleep 10 && exec bin/maxwell
  --host=mysql
  --user=maxwell
  --password=maxwellpass
  --producer=stdout
  --schema_database=maxwell
  --client_id=maxwell-stdout
  --replica_server_id=1001"
```

#### 必要な権限

```sql
-- Maxwell用のスキーマ保存DB
CREATE DATABASE IF NOT EXISTS maxwell;

-- 必要な権限を付与
GRANT ALL ON maxwell.* TO 'maxwell'@'%';
GRANT SELECT, REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'maxwell'@'%';
```

- `REPLICATION SLAVE`: binlog購読に必要
- `REPLICATION CLIENT`: `SHOW MASTER STATUS` 等の実行に必要
- `maxwell.*` への権限: スキーマ・ポジション情報の保存に必要

---

## 動作検証

### 起動

```bash
docker compose up -d
docker compose logs -f maxwell-stdout
```

起動時に以下のようなログが出れば成功です。

```
BinlogConnectorReplicator - Setting initial binlog pos to: mysql-bin.000003:xxx
BinaryLogClient - Connected to mysql:3306 at mysql-bin.000003/xxx
BinlogConnectorReplicator - Binlog connected.
```

### INSERT

```sql
INSERT INTO users (name, email, status) 
VALUES ('山田次郎', 'yamada@example.com', 'pending');
```

**出力されるJSON（例）:**

```json
{
  "database": "testdb",
  "table": "users",
  "type": "insert",
  "ts": 1769010055,
  "xid": 154,
  "commit": true,
  "data": {
    "id": 3,
    "name": "山田次郎",
    "email": "yamada@example.com",
    "status": "pending",
    "created_at": "2026-01-21 10:00:55",
    "updated_at": "2026-01-21 10:00:55"
  }
}
```

### UPDATE

```sql
UPDATE users SET status = 'active' WHERE email = 'yamada@example.com';
```

**出力されるJSON（例）:**

```json
{
  "database": "testdb",
  "table": "users",
  "type": "update",
  "ts": 1769010057,
  "xid": 365,
  "commit": true,
  "data": {
    "id": 3,
    "name": "山田次郎",
    "email": "yamada@example.com",
    "status": "active",
    "created_at": "2026-01-21 10:00:55",
    "updated_at": "2026-01-21 10:00:57"
  },
  "old": {
    "status": "pending",
    "updated_at": "2026-01-21 10:00:55"
  }
}
```

`old`フィールドに**変更前の値（差分のみ）** が含まれます。これは `binlog-row-image=FULL` 設定により取得できます。

### DELETE

```sql
DELETE FROM users WHERE email = 'yamada@example.com';
```

**出力されるJSON（例）:**

```json
{
  "database": "testdb",
  "table": "users",
  "type": "delete",
  "ts": 1769010061,
  "xid": 551,
  "commit": true,
  "data": {
    "id": 3,
    "name": "山田次郎",
    "email": "yamada@example.com",
    "status": "active",
    "created_at": "2026-01-21 10:00:55",
    "updated_at": "2026-01-21 10:00:57"
  }
}
```

---

## JSONフォーマットの比較

### Maxwell（シンプル）

```json
{
  "database": "testdb",
  "table": "users",
  "type": "insert",
  "ts": 1769010055,
  "data": { "id": 3, "name": "山田次郎", ... }
}
```

### Debezium（リッチ）

```json
{
  "schema": { /* 型情報... */ },
  "payload": {
    "before": null,
    "after": { "id": 3, "name": "山田次郎", ... },
    "source": { "connector": "mysql", ... },
    "op": "c",
    "ts_ms": 1769010055000
  }
}
```

| 項目 | Maxwell | Debezium |
|------|---------|----------|
| 構造 | フラット | ネスト（schema + payload） |
| 変更前データ | `old`（差分のみ） | `before`（全カラム） |
| 変更後データ | `data` | `after` |
| 操作タイプ | `type`: insert/update/delete | `op`: c/u/d/r |
| タイムスタンプ | `ts`（Unix秒） | `ts_ms`（ミリ秒） |
| 型情報 | 出力に含まない | 出力に含む |

**補足**: Maxwellは内部的に `maxwell.schemas` テーブルにMySQLのスキーマをコピーして保持しています。これにより、binlogには含まれない「カラム名」をJSONに付与できています。Debeziumとの違いは「出力データに型定義（メタデータ）を含めるかどうか」という点です。Schema Registryとの連携が必要な場合はDebeziumが有利です。

---

## MySQLからのレプリカ認識

Maxwellは内部的にMySQLレプリカとして振る舞い、binlogを購読します。

### SHOW PROCESSLIST

```sql
SHOW PROCESSLIST;
```

```
+----+---------+------------------+---------+-------------+------+-----------------------------------------------------------------+------+
| Id | User    | Host             | db      | Command     | Time | State                                                           | Info |
+----+---------+------------------+---------+-------------+------+-----------------------------------------------------------------+------+
| 26 | maxwell | 172.24.0.3:46586 | NULL    | Binlog Dump |  188 | Source has sent all binlog to replica; waiting for more updates | NULL |
+----+---------+------------------+---------+-------------+------+-----------------------------------------------------------------+------+
```

`Command: Binlog Dump` としてMaxwellがbinlogを購読していることが確認できます。

### ポジション管理

```sql
SELECT * FROM maxwell.positions\G
```

```
*************************** 1. row ***************************
         server_id: 1
       binlog_file: mysql-bin.000003
   binlog_position: 153736
          gtid_set: NULL
         client_id: maxwell-client-1
      heartbeat_at: NULL
last_heartbeat_read: 1768864300825
```

Debeziumは Kafka Connect の内部 offset で管理しますが、Maxwellは専用DBテーブルで管理するためMySQL側から直接確認できます。これは運用時のデバッグで便利です。

---

## 出力先の選択肢

今回はstdoutを使いましたが、Maxwellは複数の出力先に対応しています。

| Producer | 用途 |
|----------|------|
| stdout | 開発・検証 |
| kafka | Kafkaトピックへ直接出力 |
| kinesis | AWS Kinesisへ直接出力 |
| sqs | AWS SQSへ出力 |
| rabbitmq | RabbitMQへ出力 |
| redis | Redis/Valkeyへ出力 |
| pubsub | Google Cloud Pub/Subへ出力 |

Kafkaを使う場合でも、**Kafka Connectを経由せず直接Kafkaに書き込める**のがDebeziumとの大きな違いです。

### Valkey（Redis）への出力例

検証用リポジトリでは、Docker Composeのprofilesを使ってValkey出力も試せます。

```bash
# Valkeyプロファイルを有効にして起動
docker compose --profile valkey up -d

# Valkeyに格納されたデータを確認
docker exec maxwell-valkey-server valkey-cli LRANGE maxwell 0 -1
```

```json
{"database":"testdb","table":"users","type":"insert","ts":1769010055,...,"data":{"id":3,"name":"山田次郎",...}}
```

---

## まとめ

### Maxwellの特徴

- ✅ **セットアップがシンプル** - Dockerイメージ1つで完結
- ✅ **JSONが軽量** - 型情報を含まないフラットな構造
- ✅ **出力先が柔軟** - Kafka以外にも直接出力可能
- ✅ **ポジション管理が透明** - MySQLテーブルで確認可能
- ✅ **初期ロード対応** - `maxwell-bootstrap` で既存データも転送可能

### Debeziumとの使い分け

| 要件 | 推奨 |
|------|------|
| MySQL単体 + シンプルなCDC | Maxwell |
| 複数DB対応が必要 | Debezium |
| Kafka Connectエコシステムを活用 | Debezium |
| Schema Registry連携 | Debezium |
| Kinesis/SQSに直接出力 | Maxwell |
| 軽量に始めたい | Maxwell |

---

## 参考リンク

- [Maxwell's Daemon - GitHub](https://github.com/zendesk/maxwell)
- [Maxwell Documentation](https://maxwells-daemon.io/)
- [Maxwell Bootstrap（初期ロード）](https://maxwells-daemon.io/bootstrapping/)
- [検証用リポジトリ](https://github.com/toshiro3/maxwell-cdc-demo)
- [Debezium CDC入門](https://zenn.dev/toshiro3/articles/debezium-cdc-introduction)
