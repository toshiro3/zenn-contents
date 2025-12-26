---
title: "Docker不要！uv + PyIcebergでApache Icebergのメタデータ構造を理解する"
emoji: "🧊"
type: "tech"
topics: ["iceberg", "python", "uv", "datalake", "pyiceberg"]
published: true
---

## はじめに

Apache Icebergは、データレイク上でデータウェアハウスのような機能を実現するオープンテーブルフォーマットです。AWS S3 Tables、Snowflake、BigQueryなど主要クラウドサービスでも採用が進んでいるようです。

この記事では、「まずはIcebergの仕組みを軽く理解したい」ということを優先して、**Docker不要**で、**uv + PyIceberg**だけを使ってIcebergのメタデータ構造を理解することを目指します。

### 対象読者

- Apache Icebergの仕組みを理解したい方
- Docker環境なしで軽量にIcebergを試したい方
- S3 TablesやAthenaを使う前にIcebergの基礎を押さえたい方
- uvを試してみたい方

### この記事で学べること

- Icebergのメタデータ構造（metadata.json、マニフェストリスト、マニフェストファイル）
- データ追加・更新・削除時のファイル変化
- スナップショットとタイムトラベルの仕組み

## 環境構築

### uvのインストール

uvはRust製の高速なPythonパッケージマネージャーです。pip + venv + pyenvの機能を統合しており、依存解決も高速です。

```bash
# macOS (Homebrew)
brew install uv

# または公式インストーラー
curl -LsSf https://astral.sh/uv/install.sh | sh
```

インストール確認：

```bash
$ which uv
/opt/homebrew/bin/uv  # Homebrewでインストールした場合
```

### プロジェクトの作成

```bash
mkdir iceberg-tutorial
cd iceberg-tutorial

# プロジェクト初期化
uv init

# Pythonバージョン指定（必要に応じてuvが自動ダウンロード）
uv python pin 3.12

# PyIcebergと依存パッケージを追加
uv add "pyiceberg[pyarrow,sql-sqlite]"
uv add pandas
```

環境が正しく構築されたか確認します：

```bash
$ uv run python --version
Using CPython 3.12.8 interpreter at: /opt/homebrew/opt/python@3.12/bin/python3.12
Creating virtual environment at: .venv
Python 3.12.8
```

これで`pyproject.toml`と`uv.lock`が生成され、依存関係が解決されます。

## Icebergのメタデータ構造（概要）

本題に入る前に、Icebergのメタデータ構造を簡単に説明します。

```
┌─────────────────────────────────────┐
│  カタログ層                          │  ← 「現在のメタデータはどこか」を管理
│  (Glue Data Catalog, SQLite等)      │
├─────────────────────────────────────┤
│  メタデータ層                        │  ← スナップショット、スキーマ情報
│  (metadata.json, manifest list/file)│
├─────────────────────────────────────┤
│  データ層                            │  ← 実際のParquetファイル
│  (Parquetファイル群)                 │
└─────────────────────────────────────┘
```

AWSでいえば、カタログ層がGlue Data Catalog、データ層がS3に対応します。今回はローカル環境でSQLiteをカタログとして使用します。

## ハンズオン

ここからはPython REPLで一つずつ確認しながら進めます。

```bash
uv run python
```

### Step 1: インポートとカタログ設定

```python
from pyiceberg.catalog import load_catalog
from pyiceberg.schema import Schema
from pyiceberg.types import StringType, LongType, NestedField
from pathlib import Path

# warehouseディレクトリの準備
warehouse_path = Path("./warehouse").absolute()
warehouse_path.mkdir(exist_ok=True)
print(f"Warehouse: {warehouse_path}")
```

```
Warehouse: /Users/yourname/iceberg-tutorial/warehouse
```

### Step 2: カタログの作成

```python
catalog = load_catalog(
    "local",
    **{
        "type": "sql",
        "uri": "sqlite:///iceberg_catalog.db",
        "warehouse": f"file://{warehouse_path}"
    }
)

print(catalog)
```

```
local (<class 'pyiceberg.catalog.sql.SqlCatalog'>)
```

この時点で、プロジェクト直下に`iceberg_catalog.db`（SQLiteファイル）が作成されます。これがカタログとして「どのテーブルがどこにあるか」を管理します。

### Step 3: ネームスペースの作成

```python
# 既存のネームスペース確認
print(catalog.list_namespaces())

# ネームスペース作成
catalog.create_namespace("tutorial")

# 作成後の確認
print(catalog.list_namespaces())
```

```
[]
[('tutorial',)]
```

### Step 4: スキーマ定義

```python
schema = Schema(
    NestedField(1, "user_id", LongType(), required=True),
    NestedField(2, "name", StringType(), required=True),
    NestedField(3, "email", StringType(), required=False),
)

# スキーマの中身を確認
for field in schema.fields:
    print(f"  {field.field_id}: {field.name} ({field.field_type}) required={field.required}")
```

```
  1: user_id (long) required=True
  2: name (string) required=True
  3: email (string) required=False
```

`required=True`のフィールドはNULLを許容しません。これは後のデータ追加時に重要になります。

### Step 5: テーブル作成

```python
table = catalog.create_table("tutorial.users", schema=schema)
print(table)
```

```
users(
  1: user_id: required long,
  2: name: required string,
  3: email: optional string
)
```

### Step 6: メタデータの確認

```python
metadata = table.metadata

print(f"テーブルUUID: {metadata.table_uuid}")
print(f"フォーマットバージョン: {metadata.format_version}")
print(f"ロケーション: {metadata.location}")
```

```
テーブルUUID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
フォーマットバージョン: 2
ロケーション: file:///Users/yourname/iceberg-tutorial/warehouse/tutorial/users
```

### Step 7: ファイル構造の確認

この時点で別ターミナルから`warehouse`ディレクトリを確認してみましょう。

```bash
$ tree warehouse
warehouse
└── tutorial
    └── users
        └── metadata
            └── 00000-xxxxxxxx.metadata.json
```

テーブル作成直後は、`metadata`ディレクトリに最初の`metadata.json`だけが存在します。まだデータを追加していないので`data`ディレクトリはありません。

### Step 8: metadata.jsonの中身を確認

```python
import json

metadata_dir = warehouse_path / "tutorial" / "users" / "metadata"
metadata_files = sorted(metadata_dir.glob("*.metadata.json"))
print(f"メタデータファイル: {[f.name for f in metadata_files]}")

# metadata.jsonを読む
with open(metadata_files[-1]) as f:
    meta = json.load(f)

print(json.dumps(meta, indent=2)[:500])  # 長いので先頭部分だけ
```

`metadata.json`にはスキーマ定義、パーティション仕様、スナップショット履歴などが記録されています。

## データの追加

### Step 9: PyArrowでデータを準備して追加

PyIcebergでデータを追加する際は、PyArrowのTableを使用します。

```python
import pyarrow as pa

# スキーマを明示的に定義（required/optionalを合わせる）
arrow_schema = pa.schema([
    pa.field("user_id", pa.int64(), nullable=False),
    pa.field("name", pa.string(), nullable=False),
    pa.field("email", pa.string(), nullable=True),
])

data = pa.table({
    "user_id": [1, 2, 3],
    "name": ["Alice", "Bob", "Charlie"],
    "email": ["alice@example.com", "bob@example.com", None],
}, schema=arrow_schema)

table.append(data)
print("3件追加しました")
```

:::message alert
PyArrowでテーブルを作成する際、スキーマを明示的に指定しないと全カラムが`nullable=True`（optional）になります。Icebergテーブルのスキーマで`required=True`としたカラムがある場合、スキーマ不一致エラーが発生します。

```
ValueError: Mismatch in fields:
┃    ┃ Table field               ┃ Dataframe field           ┃
│ ❌ │ 1: user_id: required long │ 1: user_id: optional long │
```

このエラーが出た場合は、`pa.schema()`でスキーマを明示的に定義してください。
:::

### Step 10: スナップショットの確認

```python
table.refresh()  # メタデータ再読み込み

for snapshot in table.metadata.snapshots:
    print(f"Snapshot ID: {snapshot.snapshot_id}")
    print(f"  Operation: {snapshot.summary.get('operation')}")
    print(f"  Added records: {snapshot.summary.get('added-records')}")
```

```
Snapshot ID: 5815774430499421792
  Operation: Operation.APPEND
  Added records: 3
```

### Step 11: 追加データでスナップショットを増やす

```python
more_data = pa.table({
    "user_id": [4, 5],
    "name": ["David", "Eve"],
    "email": ["david@example.com", "eve@example.com"],
}, schema=arrow_schema)

table.append(more_data)
table.refresh()

print(f"スナップショット数: {len(table.metadata.snapshots)}")
```

```
スナップショット数: 2
```

### Step 12: ファイル構造の変化を確認

別ターミナルで`warehouse`ディレクトリを確認すると、データ追加によってファイルが増えていることがわかります。

```bash
$ tree warehouse
warehouse
└── tutorial
    └── users
        ├── data
        │   ├── 00000-0-xxxxxxxx.parquet
        │   └── 00000-0-yyyyyyyy.parquet
        └── metadata
            ├── 00000-xxxxxxxx.metadata.json
            ├── 00001-xxxxxxxx.metadata.json
            ├── 00002-xxxxxxxx.metadata.json
            ├── xxxxxxxx-m0.avro
            ├── yyyyyyyy-m0.avro
            ├── snap-xxxx-0-xxxxxxxx.avro
            └── snap-yyyy-0-yyyyyyyy.avro
```

## ファイルの役割

Step 12で確認したファイル構造について、それぞれの役割を説明します。

| ファイル種別 | 説明 |
|-------------|------|
| `*.parquet` | 実データ（カラムナフォーマット） |
| `*.metadata.json` | テーブル状態のスナップショット。操作ごとに新規作成 |
| `snap-*.avro` | マニフェストリスト。スナップショットに含まれるマニフェストファイルの一覧 |
| `*-m0.avro` | マニフェストファイル。Parquetファイルの一覧と統計情報 |

階層構造：

```
metadata.json（現在の状態）
    ↓ 参照
snap-*.avro（マニフェストリスト）
    ↓ 参照
*-m0.avro（マニフェストファイル）
    ↓ 参照
data/*.parquet（実データ）
```

## レコードの更新

### Step 13: 現在のデータを確認

```python
df = table.scan().to_pandas()
print(df)
```

```
   user_id     name              email
0        1    Alice  alice@example.com
1        2      Bob    bob@example.com
2        3  Charlie               None
3        4    David  david@example.com
4        5      Eve    eve@example.com
```

### Step 14: レコードの更新

PyIcebergでは`overwrite`を使って条件に合うデータを置き換えます。

```python
# user_id=2 (Bob) のemailを更新
updated_data = pa.table({
    "user_id": [2],
    "name": ["Bob"],
    "email": ["bob.new@example.com"],
}, schema=arrow_schema)

table.overwrite(updated_data, overwrite_filter="user_id == 2")
print("更新完了")
```

### Step 15: 更新後のデータとメタデータを確認

```python
table.refresh()
df = table.scan().to_pandas()
print(df)
```

```
   user_id     name                email
0        2      Bob  bob.new@example.com
1        1    Alice    alice@example.com
2        3  Charlie                 None
3        4    David    david@example.com
4        5      Eve      eve@example.com
```

Bobのemailが更新されています。

```python
for snap in table.metadata.snapshots:
    print(f"Snapshot ID: {snap.snapshot_id}")
    print(f"  Operation: {snap.summary.get('operation')}")
    print(f"  Added files: {snap.summary.get('added-data-files', 0)}")
    print(f"  Deleted files: {snap.summary.get('deleted-data-files', 0)}")
    print()
```

```
Snapshot ID: 5815774430499421792
  Operation: Operation.APPEND
  Added files: 1
  Deleted files: None

Snapshot ID: 3879896634347338753
  Operation: Operation.APPEND
  Added files: 1
  Deleted files: None

Snapshot ID: 5929255525955454062
  Operation: Operation.OVERWRITE
  Added files: 1
  Deleted files: 1

Snapshot ID: 7234342154326247462
  Operation: Operation.APPEND
  Added files: 1
  Deleted files: None
```

`overwrite`操作では、対象データを含むファイルが論理削除（`Deleted files: 1`）され、新しいファイルが追加（`Added files: 1`）されます。これが**Copy-on-Write**方式です。

:::message
Bobだけを更新したつもりでも、同じファイルに含まれていたAlice、Bob、Charlieの3レコードすべてが論理削除としてカウントされます。ただし、AliceとCharlieは新しいファイルに再書き込みされるため、実際にデータが失われるわけではありません。削除時のスナップショットで`deleted-records`を確認すると、この数値を確認できます。
:::

## レコードの削除

### Step 16: レコードの削除

PyIcebergでは`delete`メソッドで条件に合うレコードを削除できます。

```python
# user_id=3 (Charlie) を削除
table.delete(delete_filter="user_id == 3")
print("削除完了")
```

### Step 17: 削除後のデータとメタデータを確認

```python
table.refresh()
df = table.scan().to_pandas()
print(df)
```

```
   user_id   name                email
0        1  Alice    alice@example.com
1        2    Bob  bob.new@example.com
2        4  David    david@example.com
3        5    Eve      eve@example.com
```

Charlieが削除されています。

```python
for snap in table.metadata.snapshots:
    print(f"Snapshot ID: {snap.snapshot_id}")
    print(f"  Operation: {snap.summary.get('operation')}")
    print(f"  Added files: {snap.summary.get('added-data-files', 0)}")
    print(f"  Deleted files: {snap.summary.get('deleted-data-files', 0)}")
    print(f"  Deleted records: {snap.summary.get('deleted-records', 0)}")
    print()
```

```
Snapshot ID: 5815774430499421792
  Operation: Operation.APPEND
  Added files: 1
  Deleted files: None
  Deleted records: None

Snapshot ID: 3879896634347338753
  Operation: Operation.APPEND
  Added files: 1
  Deleted files: None
  Deleted records: None

Snapshot ID: 5929255525955454062
  Operation: Operation.OVERWRITE
  Added files: 1
  Deleted files: 1
  Deleted records: 3

Snapshot ID: 7234342154326247462
  Operation: Operation.APPEND
  Added files: 1
  Deleted files: None
  Deleted records: None

Snapshot ID: 4494934084334135847
  Operation: Operation.OVERWRITE
  Added files: 1
  Deleted files: 1
  Deleted records: 2
```

削除操作は`Operation.OVERWRITE`として記録され、`deleted-records`と`deleted-data-files`が増加しています。Icebergでは削除も内部的にはファイルの書き換え（Copy-on-Write）として処理されます。

3番目のスナップショット（`Deleted records: 3`）はStep 14での更新時、5番目のスナップショット（`Deleted records: 2`）は今回の削除時のものです。

:::message
更新時と同様に、`Deleted records` は「ファイル単位で論理削除されたレコード数」です。Charlieだけを削除しても、同じファイルに含まれていた他のレコードも論理削除としてカウントされます。削除対象以外のレコードは新しいファイルに再書き込みされます。
:::

## タイムトラベル

Icebergの大きな特徴の一つが、過去のスナップショットにアクセスできる「タイムトラベル」機能です。

### Step 18: 過去のスナップショットでデータを読む

```python
# 最初のスナップショットのIDを取得
first_snapshot_id = table.metadata.snapshots[0].snapshot_id

# 過去のスナップショットでスキャン
old_df = table.scan(snapshot_id=first_snapshot_id).to_pandas()
print("=== 最初のappend時点のデータ ===")
print(old_df)
```

```
=== 最初のappend時点のデータ ===
   user_id     name              email
0        1    Alice  alice@example.com
1        2      Bob    bob@example.com
2        3  Charlie               None
```

更新前のBobのメールアドレスや、削除したCharlieのレコードが確認できます。

## まとめ

この記事では、uv + PyIcebergを使ってApache Icebergのメタデータ構造を確認しました。

### 学んだこと

1. **カタログ**: テーブルの場所を管理（今回はSQLite、本番ではGlue Data Catalogなど）
2. **metadata.json**: テーブルの状態を記録。操作ごとに新しいファイルが作成される
3. **マニフェストリスト/ファイル**: データファイルの一覧と統計情報を階層的に管理
4. **スナップショット**: 各時点のテーブル状態。タイムトラベルの基盤
5. **Copy-on-Write**: 更新・削除時は元ファイルを残して新ファイルを作成

### 次のステップ

- **AWS S3 Tables**: Icebergをマネージドで使えるAWSサービス
- **dbt-athena**: dbtからAthena経由でIcebergテーブルを操作
- **パーティショニング**: Hidden Partitioningによる柔軟なパーティション管理

Icebergの内部構造を理解しておくと、S3 TablesやAthenaを使う際にも「裏で何が起きているか」がわかり、トラブルシューティングや設計判断に役立つのではないかと思います。

## 参考資料

- [Apache Iceberg公式ドキュメント](https://iceberg.apache.org/docs/latest/)
- [PyIceberg公式ドキュメント](https://py.iceberg.apache.org/)
- [uv公式ドキュメント](https://docs.astral.sh/uv/)
