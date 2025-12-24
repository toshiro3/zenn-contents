---
title: "DynamoDBのデータをAthenaで分析可能にする構成を検証してみた（dbt・S3 Tables も試した結果）"
emoji: "🔍"
type: "tech"
topics: ["aws", "dynamodb", "athena", "dbt", "s3"]
published: false
---

## はじめに

DynamoDBに保存されたデータを分析したいけど、柔軟な検索ができなくて困っている——そんな課題に直面したことはありませんか？

本記事では、DynamoDBのデータをAthenaで検索・分析可能にするための構成を検証した結果をまとめます。dbtやS3 Tablesといった選択肢も試した上で、最終的にシンプルな構成に落ち着いた経緯と、そこから得られた知見を共有します。

### 対象読者

- DynamoDBのデータを分析したいと考えている方
- 分析基盤の構築を検討している方
- dbt + Athenaの構成に興味がある方
- S3 Tablesの現状を知りたい方

### 検証環境

- AWS（Athena、S3、DynamoDB、Glue）
- dbt-athena-community 1.9.5
- Python 3.11

## 背景：DynamoDBの検索制限

今回の検証の背景として、あるサービスでユーザーのアクティビティログをDynamoDBに保存していました。レコード数が多くなることと、書き込み頻度が高くなることを想定してDynamoDBを選定したのですが、以下の課題が発生しました。

### DynamoDBでできること・できないこと

| できること | できないこと |
|-----------|-------------|
| パーティションキー + ソートキーでの高速Query | 柔軟な条件検索（WHERE句的な） |
| GSI（グローバルセカンダリインデックス）での別軸検索 | 複数条件のAND/OR検索 |
| 大量の書き込み | 集計・分析クエリ |
| 自動スケーリング | JOINやサブクエリ |

「日別のアクティビティ数を集計したい」「特定ユーザーの行動を時系列で見たい」といった分析ニーズに対応できず、データを分析可能にする方法を検討することになりました。

## 検証した構成

以下の構成を段階的に検証しました。

```
【検証した構成】
DynamoDB → S3 → Athena → dbt → Iceberg

【最終的に採用した構成】
DynamoDB → S3（JSON Lines） → Athena
```

## Step 1: DynamoDBからS3への抽出

### 抽出方法の選択肢

| 方法 | 特徴 | 適したケース |
|------|------|-------------|
| DynamoDB Export（フル） | テーブル全体を一括出力 | 初回ロード、リセット |
| DynamoDB Export（増分） | 指定期間の変更を出力 | 日次バッチ |
| DynamoDB Streams + Firehose | リアルタイムにS3へ | ニアリアルタイム要件 |
| 独自スクリプト | 柔軟にカスタマイズ可能 | 特定条件での抽出 |

今回は検証目的のため、**独自のPythonスクリプト**でパーティションキーを指定して抽出する方式を採用しました。

### 抽出スクリプト

```python
import argparse
import boto3
import json
from decimal import Decimal

class DecimalEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, Decimal):
            return int(obj) if obj % 1 == 0 else float(obj)
        return super().default(obj)

def get_items_by_partition_key(table, pk_name: str, pk_value) -> list:
    """パーティションキーを指定してアイテムを取得"""
    items = []
    response = table.query(
        KeyConditionExpression=f'{pk_name} = :pk',
        ExpressionAttributeValues={':pk': pk_value}
    )
    items.extend(response['Items'])
    
    while 'LastEvaluatedKey' in response:
        response = table.query(
            KeyConditionExpression=f'{pk_name} = :pk',
            ExpressionAttributeValues={':pk': pk_value},
            ExclusiveStartKey=response['LastEvaluatedKey']
        )
        items.extend(response['Items'])
    
    return items

def save_to_s3(s3_client, bucket: str, key: str, data: list):
    """JSON Lines形式でS3に保存"""
    lines = [json.dumps(item, ensure_ascii=False, cls=DecimalEncoder) for item in data]
    json_lines = '\n'.join(lines)
    s3_client.put_object(
        Bucket=bucket,
        Key=key,
        Body=json_lines.encode('utf-8'),
        ContentType='application/json'
    )
```

### ポイント：JSON Lines形式

最初、JSONを配列形式で出力していたのですが、Athenaでクエリするとエラーになりました。調べてみると、AthenaでJSONを読む場合は**JSON Lines形式**（1行1レコード）にする必要があることがわかりました。

> 参考: [Best practices for reading JSON data - Amazon Athena](https://docs.aws.amazon.com/athena/latest/ug/parsing-json-data.html)

```json
// ❌ 配列形式（Athenaで読めない）
[
  {"id": 1, "name": "Alice"},
  {"id": 2, "name": "Bob"}
]

// ✅ JSON Lines形式（Athenaで読める）
{"id": 1, "name": "Alice"}
{"id": 2, "name": "Bob"}
```

### パーティション設計

S3上のデータは日次でパーティションを切りました。

```
s3://bucket/raw/activity_logs/
  └── dt=2024-07-17/
        └── partition_001.json
  └── dt=2024-07-18/
        └── partition_002.json
```

## Step 2: AthenaでExternal Table作成

### パーティションプロジェクションの活用

通常、S3にファイルを追加するたびに `MSCK REPAIR TABLE` でパーティションを認識させる必要があります。これを自動化するのが**パーティションプロジェクション**です。

```sql
CREATE EXTERNAL TABLE raw_activity_logs (
    user_id bigint,
    activity_type string,
    created_at bigint,
    metadata string
)
PARTITIONED BY (dt string)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
LOCATION 's3://your-bucket/raw/activity_logs/'
TBLPROPERTIES (
    'projection.enabled' = 'true',
    'projection.dt.type' = 'date',
    'projection.dt.range' = '2024-01-01,NOW',
    'projection.dt.format' = 'yyyy-MM-dd',
    'projection.dt.interval' = '1',
    'projection.dt.interval.unit' = 'DAYS',
    'storage.location.template' = 's3://your-bucket/raw/activity_logs/dt=${dt}'
)
```

これにより、ファイルを追加するだけで自動的にクエリ可能になります。

## Step 3: dbt + Athenaの検証

### dbtとは

dbt（data build tool）は、SQLベースでデータ変換パイプラインを管理するツールです。

- SQLでモデル（変換ロジック）を定義
- モデル間の依存関係を自動解決
- テスト、ドキュメント生成機能

### セットアップ

```bash
pip install dbt-athena-community
dbt init analytics
```

`~/.dbt/profiles.yml` の設定例：

```yaml
analytics:
  outputs:
    dev:
      type: athena
      s3_staging_dir: s3://your-bucket/athena-results/
      s3_data_dir: s3://your-bucket/dbt-data/
      region_name: ap-northeast-1
      schema: analytics
      database: awsdatacatalog
      threads: 4
      aws_profile_name: your-profile
  target: dev
```

### モデルの作成

#### staging層（View）

```sql
-- models/staging/stg_activity_logs.sql
with source as (
    select * from {{ source('raw', 'raw_activity_logs') }}
),

transformed as (
    select
        user_id,
        activity_type,
        created_at,
        from_unixtime(created_at) as activity_at,
        metadata,
        dt as partition_date
    from source
)

select * from transformed
```

#### mart層（Icebergテーブル）

```sql
-- models/marts/mart_activity_logs.sql
{{
    config(
        materialized='table',
        table_type='iceberg',
        format='parquet',
        partitioned_by=['partition_month']
    )
}}

with stg as (
    select * from {{ ref('stg_activity_logs') }}
)

select
    user_id,
    activity_type,
    created_at,
    activity_at,
    partition_date,
    date_format(activity_at, '%Y-%m') as partition_month
from stg
```

### 検証結果

dbtは正常に動作し、staging（View）→ mart（Iceberg）の変換パイプラインを構築できました。

しかし、以下の課題から**採用を見送り**ました。

| 課題 | 詳細 |
|------|------|
| 抽出は別実装 | dbtはTransformのみ。E（抽出）は自前で実装が必要 |
| インクリメンタルの複雑さ | 条件設計、リカバリ手順が煩雑 |
| 再集計の手間 | ソースデータ修正時は `--full-refresh` が必要 |
| 小規模だとオーバーヘッド | 設定ファイルや学習コストが負担 |

#### dbtが向いているケース

- チーム開発（SQLベースでレビューしやすい）
- 変換ロジックが複雑（数十〜数百のモデル）
- 本番運用で品質担保が必要（テスト、CI/CD連携）

今回のような小規模な検証・分析用途では、シンプルな構成の方が運用しやすいと判断しました。

## Step 4: S3 Tablesの検証

### S3 Tablesとは

AWS re:Invent 2024で発表された新機能で、Apache Iceberg形式のテーブルをマネージドで管理できます。

期待される機能：
- コンパクションの自動化
- スナップショット管理の自動化
- ガベージコレクションの自動化

### 検証した構成

dbt-athenaを使って、以下の構成でS3 Tablesへの出力を試みました。

```
raw層（通常S3 / JSON Lines）
    ↓ dbt-athena
staging層（S3 Tables / VIEW）
    ↓ dbt-athena
mart層（S3 Tables / TABLE）
```

### 検証結果

:::message alert
2025年12月時点での検証結果です。今後のアップデートで状況が変わる可能性があります。
:::

dbt-athenaでS3 Tablesカタログを出力先に指定したところ、カタログの認識段階で失敗しました。

```
An error occurred (InvalidRequestException) when calling the GetDataCatalog operation: 
DataCatalog s3tablescatalog/bucket-name was not found.
```

dbt-athenaは内部でAthenaの `GetDataCatalog` APIを使用してカタログ情報を取得しますが、S3 TablesのカタログはこのAPIでは取得できません。S3 Table Bucketが存在する状態でも同様のエラーとなりました。

**結論**: 現時点ではdbt-athenaとS3 Tablesの組み合わせは利用できないようです。

### 補足：dbtを使わない場合

dbt-athenaを介さずに直接AthenaやAWSマネージドサービス（Glue、EventBridge Pipesなど）からS3 Tablesへデータを投入する場合は、この制限には該当しません。

ただし、AthenaコンソールからS3 Tablesカタログに対して `CREATE VIEW` を実行すると、以下のエラーが発生します。

```
CREATE VIEW statements are not yet supported for Cross-Account Glue DataCatalogs.
```

S3 TablesはFederated Catalog（クロスアカウントカタログ扱い）として認識されるため、VIEW作成には制限があります。TABLE作成やデータ投入（INSERT）は問題なく実行できました。

> 参考: [Register S3 table bucket catalogs and query Tables from Athena](https://docs.aws.amazon.com/athena/latest/ug/gdc-register-s3-table-bucket-cat.html)

### 通常S3 + Icebergとの比較

dbt-athenaでS3 Tablesを使えない場合、通常のS3バケット上にIceberg形式でテーブルを作成することになります。この場合、コンパクション等は自前で実行する必要があります。

```sql
-- コンパクション
OPTIMIZE your_table REWRITE DATA USING BIN_PACK

-- 古いスナップショット削除
VACUUM your_table
```

実際にdbt-athenaで作成したIcebergテーブルに対してこれらのコマンドを試したところ、記事執筆時点（2025年12月）では**Athenaコンソールのクエリエディタからは実行できませんでした**。

```
line 1:1: mismatched input 'OPTIMIZE'. Expecting: 'ALTER', 'ANALYZE', ...
```

一方、**AWS CLIからは正常に実行できました**。

```bash
aws athena start-query-execution \
  --query-string "OPTIMIZE your_db.your_table REWRITE DATA USING BIN_PACK" \
  --result-configuration OutputLocation=s3://your-bucket/athena-results/ \
  --work-group primary \
  --profile your-profile
```

今回の検証では、そこまでの運用負荷をかける必要はないと判断し、**シンプルにJSON Lines形式のままAthenaでクエリする構成**を採用しました。

## 最終的な構成

検証の結果、以下のシンプルな構成を採用しました。

```
DynamoDB
    ↓ 抽出スクリプト（Python）
S3（JSON Lines / 日次パーティション）
    ↓ パーティションプロジェクション
Athena External Table
```

### この構成のメリット

- **シンプル**: 構成要素が少なく、理解しやすい
- **運用負荷が低い**: ファイル追加だけでクエリ可能
- **柔軟**: 必要に応じてVIEWで変換を追加できる
- **コスト効率**: 最小限のAWSサービスで実現

### この構成のデメリット

- **パフォーマンス**: 大量データではJSON Linesは遅い（Parquetへの変換を検討）
- **リアルタイム性なし**: バッチ処理のため遅延がある
- **変換ロジックの管理**: VIEWが増えると管理が煩雑になる可能性

## 学びと知見

### 1. DynamoDB選定の振り返り

「レコード数が多い」「書き込み頻度が高い」という理由でDynamoDBを選定しました。書き込み性能の観点ではこの選定は妥当でしたが、**分析・可視化の要件に対する考慮が不足していました**。

DynamoDBが適しているケース：
- 書き込みが非常に多い
- ミリ秒レベルのレイテンシが必要
- アクセスパターンが固定されている

DynamoDBをデータストアとして採用する場合、分析・可視化の対象にするには別途の仕組み（今回検証したS3 + Athenaなど）が必要になります。設計段階でこの点を考慮しておくべきでした。

### 2. dbtの適用範囲

dbtは強力なツールですが、以下の前提がある場合に効果を発揮しそうです。

- ソースデータは基本的に正しい
- データは追加されていく（過去は変わらない）
- 変換ロジックが複雑で、チームで管理したい

小規模・単純な変換であれば、直接SQLやVIEWで十分なように思えました。

## まとめ

DynamoDBのデータをAthenaで分析可能にする構成を検証しました。

- **dbt**: 小規模では過剰、シンプルな構成を優先
- **S3 Tables**: 今回の規模では不要と判断（dbt-athena連携は記事執筆時点で未対応）
- **採用した構成**: DynamoDB → S3（JSON Lines） → Athena

「とりあえず分析できるようにしたい」という要件であれば、シンプルな構成から始めて、必要に応じて拡張していくアプローチが現実的だと感じました。

この記事が、同様の課題を抱えている方の参考になれば幸いです。
