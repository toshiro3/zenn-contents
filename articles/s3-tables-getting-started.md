---
title: "S3 Tablesを触る前に知っておきたかったこと"
emoji: "🗃️"
type: "tech"
topics: ["aws", "s3", "iceberg", "athena", "データ分析"]
published: true
---

## はじめに

AWS re:Invent 2024で発表され、2024年末にGAとなった **Amazon S3 Tables**。Apache Icebergをネイティブサポートし、分析ワークロード向けに最適化された新しいストレージサービスです。

私も触ってみようと思ったのですが、最初にいくつか勘違いをしていて手こずりました。この記事では、S3 Tablesを触る前に知っておきたかったポイントを整理します。

## 最初の勘違い：S3 Bucket と S3 Tables は1:1ではない

私は最初、既存のS3バケットに対してテーブル機能が追加されるものだと思っていました。つまり「S3 Bucket : S3 Table = 1:1」という理解です。

**これは完全に間違いでした。**

S3 Tablesでは、従来のS3バケット（General Purpose Bucket）とは**完全に別の「Table Bucket」という新しいタイプのバケット**を作成します。

## S3 Tablesの構成を理解する

S3 Tablesは以下の3層構造になっています。

```
Table Bucket（テーブル専用バケット）
  └── Namespace（論理的なグループ、DBで言うスキーマ）
        └── Table（Icebergテーブル）
```

### 従来のS3との比較

| 項目 | 従来のS3（General Purpose） | S3 Tables |
|------|---------------------------|-----------|
| バケットタイプ | General Purpose Bucket | Table Bucket |
| データ形式 | 任意のオブジェクト | Apache Iceberg形式のみ |
| 構造 | バケット > オブジェクト | バケット > Namespace > Table |
| メンテナンス | 手動（コンパクション等） | 自動（コンパクション、スナップショット管理） |
| クエリ性能 | - | 通常のIcebergより最大3倍高速 |
| トランザクション | - | 通常のS3より最大10倍高い処理能力 |

### 重要なポイント

- **Table Bucketは従来のS3バケットとは別物**：既存のバケットをTable Bucketに変換することはできません
- **Namespaceは必須**：テーブルを作成する前に、必ずNamespaceを作成する必要があります
- **Icebergフォーマット専用**：S3 TablesはApache Iceberg形式のみをサポートします

## 実際に作ってみる

それでは、AWS CLIを使ってS3 Tablesを作成してみましょう。

### 前提条件

- AWS CLIがインストールされていること
- 適切なIAM権限があること（`s3tables:*` 権限）

### 1. Table Bucketの作成

```bash
aws s3tables create-table-bucket \
  --name my-analytics-bucket
```

レスポンス例：
```json
{
  "arn": "arn:aws:s3tables:ap-northeast-1:123456789012:bucket/my-analytics-bucket",
  "name": "my-analytics-bucket"
}
```

作成されたTable Bucketを確認します。

```bash
aws s3tables list-table-buckets
```

レスポンス例：
```json
{
  "tableBuckets": [
    {
      "arn": "arn:aws:s3tables:ap-northeast-1:123456789012:bucket/my-analytics-bucket",
      "name": "my-analytics-bucket",
      "ownerAccountId": "123456789012",
      "createdAt": "2025-12-17T03:23:48.028595+00:00",
      "tableBucketId": "2c216f11-a185-4908-a756-2c25db36d893",
      "type": "customer"
    }
  ]
}
```

:::message
Table Bucketの命名規則は従来のS3バケットと異なります。
- 3〜63文字
- 小文字、数字、ハイフン（-）のみ使用可能
- **リージョン内でアカウント単位で一意であればOK**（従来のようにグローバルで一意である必要はありません）
:::

### 2. Namespaceの作成

テーブルを作成する前に、Namespaceを作成します。以降のコマンドで使用するため、ARNを変数に格納しておきます。

```bash
TABLE_BUCKET_ARN="arn:aws:s3tables:ap-northeast-1:123456789012:bucket/my-analytics-bucket"
```

```bash
aws s3tables create-namespace \
  --table-bucket-arn $TABLE_BUCKET_ARN \
  --namespace analytics
```

レスポンス例：
```json
{
  "tableBucketARN": "arn:aws:s3tables:ap-northeast-1:123456789012:bucket/my-analytics-bucket",
  "namespace": [
    "analytics"
  ]
}
```

作成されたNamespaceを確認します。

```bash
aws s3tables list-namespaces \
  --table-bucket-arn $TABLE_BUCKET_ARN
```

レスポンス例：
```json
{
  "namespaces": [
    {
      "namespace": [
        "analytics"
      ],
      "createdAt": "2025-12-17T03:33:03.240750+00:00",
      "createdBy": "123456789012",
      "ownerAccountId": "123456789012",
      "namespaceId": "ac142b85-0dc3-4f80-be6c-2dd8446c52ab",
      "tableBucketId": "2c216f11-a185-4908-a756-2c25db36d893"
    }
  ]
}
```

:::message
S3コンソールにはNamespaceの独立した一覧画面がありません。この時点で確認するにはCLIで `list-namespaces` を使用します。テーブル作成後は、Table Bucketの「Tables」タブでテーブルと一緒にNamespaceも確認できます。
:::

:::message
Namespaceの命名規則：
- 1〜255文字
- 小文字、数字、アンダースコア（_）のみ
- 先頭・末尾にアンダースコアは使用不可
- ハイフン（-）やピリオド（.）は使用不可
:::

### 3. Tableの作成

テーブル定義用のJSONファイルを作成します。

```json:mytable.json
{
  "tableBucketARN": "arn:aws:s3tables:ap-northeast-1:123456789012:bucket/my-analytics-bucket",
  "namespace": "analytics",
  "name": "page_views",
  "format": "ICEBERG",
  "metadata": {
    "iceberg": {
      "schema": {
        "fields": [
          {"name": "event_id", "type": "string", "required": true},
          {"name": "user_id", "type": "string"},
          {"name": "page_url", "type": "string"},
          {"name": "viewed_at", "type": "timestamp"},
          {"name": "duration_seconds", "type": "int"}
        ]
      }
    }
  }
}
```

```bash
aws s3tables create-table \
  --cli-input-json file://mytable.json
```

レスポンス例：
```json
{
  "tableARN": "arn:aws:s3tables:ap-northeast-1:123456789012:bucket/my-analytics-bucket/table/e4f2a9b1-51fe-49f2-a503-54d45b19aabd",
  "versionToken": "a873a2e8e114b812bcb2"
}
```

作成されたテーブルを確認します。

```bash
aws s3tables list-tables \
  --table-bucket-arn $TABLE_BUCKET_ARN
```

レスポンス例：
```json
{
  "tables": [
    {
      "namespace": [
        "analytics"
      ],
      "name": "page_views",
      "type": "customer",
      "tableARN": "arn:aws:s3tables:ap-northeast-1:123456789012:bucket/my-analytics-bucket/table/e4f2a9b1-51fe-49f2-a503-54d45b19aabd",
      "createdAt": "2025-12-17T03:46:12.442984+00:00",
      "modifiedAt": "2025-12-17T03:46:12.442984+00:00"
    }
  ]
}
```

:::message
Namespaceの一覧画面はS3コンソールにありませんが、テーブルはTable Bucketの詳細画面にある「Tables」タブから確認できます。テーブル一覧には所属するNamespaceも表示されます。
:::

## Athenaからクエリする

S3 Tablesに作成したテーブルは、Athenaから直接クエリできます。ただし、事前にAWS統合を有効にする必要があります。

### 1. AWS統合を有効にする

S3コンソールから「Table buckets」を開き、「Enable integration」ボタンをクリックします。

![Table bucketsページ](/images/s3-tables/table-buckets-list.png)
*「Enable integration」ボタンをクリック*

確認画面が表示されます。

![Enable integration確認画面](/images/s3-tables/enable-integration-dialog.png)
*統合を有効にする確認画面*

この画面で説明されている通り、統合を有効にすると以下のことが行われます：

#### Integration with AWS analytics services
- Amazon Athena、Redshift、EMR、SageMakerからTable Bucketsにアクセス可能になる
- AWS Glue Data Catalogに `s3tablescatalog` というカタログが作成される
- 各Table Bucketがサブカタログとして登録される

#### AWS Lake Formation table bucket registration
- `S3TablesRoleForLakeFormation` というIAMロールが自動作成される
- このロールにより、Lake FormationがTable Bucketsにアクセスできるようになる

#### AWS Glue catalog creation
- Glue Data Catalogに `s3tablescatalog` カタログが作成される
- リージョン内の各Table Bucketに対応するサブカタログも作成される

:::message alert
統合を有効にすると、AWS GlueとLake Formationが利用されるため、Glueのリクエスト・ストレージコストが発生する可能性があります。
:::

「Enable integration」ボタンをクリックして有効化を完了します。

### 2. Lake Formationの権限設定

管理者権限（AdministratorAccess）を持つIAMユーザーであれば、統合を有効にした時点でAthenaからクエリできます。

制限付きのIAMユーザー/ロールを使用する場合は、Lake Formationでテーブルへのアクセス権限を付与する必要があります。詳細は[公式ドキュメント](https://docs.aws.amazon.com/lake-formation/latest/dg/granting-table-permissions.html)を参照してください。

### 3. Athenaでクエリ

Athenaコンソールを開き、以下を選択します：

- **データソース**: `AWSDataCatalog`
- **カタログ**: `s3tablescatalog/my-analytics-bucket`（Table Bucket名がカタログ名になる）
- **データベース**: `analytics`（Namespace名がデータベース名になる）

```sql
-- データを挿入
INSERT INTO analytics.page_views 
VALUES 
  ('evt-001', 'user-123', '/home', TIMESTAMP '2025-01-15 10:00:00', 30),
  ('evt-002', 'user-456', '/products', TIMESTAMP '2025-01-15 10:05:00', 45);

-- クエリ実行
SELECT * FROM analytics.page_views;
```

:::message alert
Athenaのクエリ結果を保存するS3バケット（通常のGeneral Purpose Bucket）は別途必要です。これはTable Bucketとは別に用意してください。
:::

## 料金について

S3 Tablesの料金体系は従来のS3とは異なります。

### S3 Tablesストレージ料金（東京リージョン）

| 容量 | 料金 |
|------|------|
| 最初の 50 TB/月 | $0.0288/GB |
| 次の 450 TB/月 | $0.0276/GB |
| 500 TB/月以上 | $0.0265/GB |
| モニタリング | $0.025/1,000オブジェクト |

### S3 Standardとの比較

| 項目 | S3 Standard | S3 Tables |
|------|-------------|-----------|
| ストレージ（最初の50 TB/月） | $0.025/GB | $0.0288/GB |
| コンパクション | 手動（自前で実装） | 自動（$0.005/GB ※Binpackの場合） |

※東京リージョン、2025年12月時点の料金

ストレージ単価はS3 Standardの$0.025/GBに対してS3 Tablesは$0.0288/GBとやや高めですが、自動コンパクションによるクエリ性能向上（最大3倍）と運用負荷軽減を考慮すると、分析用途では十分にメリットがあります。特に、自前でIcebergのメンテナンス処理を実装・運用するコストを考えれば、マネージドで任せられる価値は大きいです。

## ハマりやすいポイントまとめ

### 1. 従来のS3バケットとは別物

- 既存のS3バケットは使えない
- Table Bucketという新しいタイプのバケットを作成する
- オブジェクトを直接PUT/GETするのではなく、テーブルとして操作する

### 2. Namespaceが必須

- テーブル作成前に必ずNamespaceを作成する
- 1つのTable Bucketに最大10,000個のNamespaceを作成可能
- Namespaceはネストできない（1階層のみ）

### 3. S3コンソールでの確認方法

- Namespaceの独立した一覧画面はS3コンソールにない
- テーブルはTable Bucketの詳細画面 →「Tables」タブから確認可能
- テーブル一覧に所属するNamespaceも表示される
- Namespace一覧をCLIで確認するには `list-namespaces` を使用
- AWS統合を有効化後は、AthenaコンソールでCatalog > Database（Namespace）> Tableとしても確認可能

### 4. AWS統合とLake Formation

- AthenaなどAWSサービスから使うには「Enable integration」で統合を有効にする
- 制限付きのIAMユーザー/ロールを使用する場合は、Lake Formationでの権限設定が必要

### 5. 用語のマッピング

S3 TablesとAWSサービスで用語が異なるので注意：

| S3 Tables | Athena/Glue | 
|-----------|-------------|
| Table Bucket | Catalog |
| Namespace | Database |
| Table | Table |

## リソースの削除

検証が終わったら、不要なコストを避けるためにリソースを削除しましょう。削除は作成と逆の順序で行います。

:::message alert
Table Bucketはコンソールから削除できません。AWS CLI、SDK、またはREST APIを使用する必要があります。
:::

### 1. Tableの削除

```bash
aws s3tables delete-table \
  --table-bucket-arn arn:aws:s3tables:ap-northeast-1:123456789012:bucket/my-analytics-bucket \
  --namespace analytics \
  --name page_views
```

### 2. Namespaceの削除

Namespace内のすべてのテーブルを削除してから、Namespaceを削除します。

```bash
aws s3tables delete-namespace \
  --table-bucket-arn arn:aws:s3tables:ap-northeast-1:123456789012:bucket/my-analytics-bucket \
  --namespace analytics
```

### 3. Table Bucketの削除

すべてのNamespaceを削除してから、Table Bucketを削除します。

```bash
aws s3tables delete-table-bucket \
  --table-bucket-arn arn:aws:s3tables:ap-northeast-1:123456789012:bucket/my-analytics-bucket
```

### 削除の確認

```bash
aws s3tables list-table-buckets
```

:::message
AWS統合を有効にした場合、Glue Data Catalogに作成された `s3tablescatalog` カタログは、Table Bucketを削除すると自動的にサブカタログも削除されます。ただし、Lake Formationで付与した権限設定は残る場合があるため、必要に応じてLake Formationコンソールから確認・削除してください。
:::

## まとめ

S3 Tablesは、分析ワークロード向けに最適化された新しいストレージオプションです。従来のS3とは構成が異なるため、最初は戸惑うかもしれませんが、以下のポイントを押さえておけばスムーズに始められます。

1. **Table Bucketは新しいバケットタイプ**（従来のS3バケットとは別物）
2. **3層構造**を理解する（Table Bucket > Namespace > Table）
3. **AWS統合**を有効にしてAthenaなどから利用可能に
4. **自動メンテナンス**でコンパクションなどが自動化される

dbtやSparkと組み合わせたデータパイプラインを構築する際にも、S3 Tablesは有力な選択肢になりそうです。次回は、実際のデータパイプラインでの活用例を紹介したいと思います。

## 参考リンク

- [Amazon S3 Tables 公式ドキュメント](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html)
- [Amazon S3 Tables の料金](https://aws.amazon.com/s3/pricing/)
- [AWS Blog: New Amazon S3 Tables](https://aws.amazon.com/jp/blogs/news/new-amazon-s3-tables-storage-optimized-for-analytics-workloads/)
