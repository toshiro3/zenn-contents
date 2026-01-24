---
title: "Dagster入門③ - Partitionでデータを分割管理する"
emoji: "🔷"
type: "tech"
topics: ["dagster", "docker", "python", "データエンジニアリング"]
published: false
---

## はじめに

[前回の記事](https://zenn.dev/toshiro3/articles/dagster-resource-schedule)では、ResourceとScheduleを使って実用的なパイプラインに近づけました。

今回は、データを日付などの単位で分割管理する**Partition**について検証していきます。

**本記事で扱う内容**
- Partitionの概念とべき等性
- 日付パーティションの実装
- 階層化したファイルパスへの出力
- バックフィルによる一括処理
- Partition Health画面での状態確認

検証用のコードはGitHubで公開しています。
https://github.com/toshiro3/workflow-orchestration-lab/tree/v3-partition

## Partitionとは

Partitionは、データを特定の単位（日付、カテゴリなど）で分割して管理する仕組みです。大量のデータを効率的に処理し、必要な部分だけを再処理できるようになります。

### なぜPartitionを使うのか

Partitionを使わない場合、データ処理は以下のような問題を抱えます。

- 全データを毎回処理する必要があり、時間がかかる
- 一部のデータだけ再処理したい場合も全体を実行する必要がある
- どの期間のデータが最新なのか把握しにくい

Partitionを使うことで、以下のメリットが得られます。

| メリット | 説明 |
|---------|------|
| 効率的な処理 | 必要なパーティションだけを処理 |
| 選択的な再処理 | 失敗したパーティションだけを再実行 |
| 状態の可視化 | どの期間が処理済みか一目でわかる |
| バックフィル | 過去データを一括で処理 |

### べき等性（Idempotency）との関係

Partitionを使う上で重要な概念が**べき等性**です。べき等性とは「何度実行しても同じ結果になる」という性質です。

今回の実装では、パーティションキー（例: `2026-01-15`）から出力先パスを決定論的に生成します。

```python
# パーティションキーから階層化したパスを生成
# 2026-01-15 → /data/2026/01/15/sales.csv
output_path = f"{base_dir}/{year}/{month}/{day}/sales.csv"
```

同じパーティションキーは常に同じパスに出力されるため、何度実行しても結果は同じファイルに上書きされます。これにより、失敗時の再実行が安全に行えます。

また、日付ごとにディレクトリを分けることで、**その日のデータを再処理しても他の日のデータに一切影響を与えない（副作用の分離）** というメリットもあります。誤って過去データを壊してしまう事故を防ぐ設計として、実務では非常に重要なポイントです。

## Partitionの種類

Dagsterでは複数のパーティション定義が用意されています。

| 種類 | 説明 | 例 |
|------|------|-----|
| `DailyPartitionsDefinition` | 日単位 | 2026-01-01, 2026-01-02, ... |
| `HourlyPartitionsDefinition` | 時間単位 | 2026-01-01-00:00, ... |
| `WeeklyPartitionsDefinition` | 週単位 | 2026-01-06（週の開始日） |
| `MonthlyPartitionsDefinition` | 月単位 | 2026-01, 2026-02, ... |
| `StaticPartitionsDefinition` | 固定値 | ["us", "jp", "eu"] |
| `DynamicPartitionsDefinition` | 動的 | 実行時に決定 |

今回は最もよく使われる`DailyPartitionsDefinition`を使用します。

## Partitionの実装

### ファイル構成

前回からファイルを追加しました。

```
my_dagster_project/
├── __init__.py
├── assets.py              # 既存（非パーティション）
├── partitions.py          # 新規：パーティション定義
├── partitioned_assets.py  # 新規：パーティション対応Asset
├── resources.py
└── schedules.py
```

### パーティション定義

`partitions.py`で日付パーティションを定義します。

```python
from dagster import DailyPartitionsDefinition

# 日付パーティションの定義（2026年1月1日から開始）
daily_partitions = DailyPartitionsDefinition(
    start_date="2026-01-01",
    timezone="Asia/Tokyo",
)
```

`start_date`から現在日までのパーティションが自動的に生成されます。`timezone`を指定することで、日本時間基準でパーティションが区切られます。

### パーティション対応Asset

`partitioned_assets.py`でパーティション対応のAssetを定義します。

```python
import os
import pandas as pd
from dagster import asset, AssetExecutionContext, MaterializeResult, MetadataValue

from .partitions import daily_partitions
from .resources import SalesDataConfig


def partition_key_to_path(partition_key: str, base_dir: str, filename: str) -> str:
    """パーティションキー（2026-01-15）を階層化したパスに変換
    
    例: 2026-01-15 → /base_dir/2026/01/15/filename
    """
    year, month, day = partition_key.split("-")
    return os.path.join(base_dir, year, month, day, filename)


@asset(
    partitions_def=daily_partitions,
    description="日付パーティションごとの売上データ",
)
def partitioned_sales_data(
    context: AssetExecutionContext,
    sales_config: SalesDataConfig,
) -> MaterializeResult:
    """日付パーティションで売上データを処理するAsset"""
    
    # パーティションキーを取得（例: "2026-01-15"）
    partition_key = context.partition_key
    context.log.info(f"Processing partition: {partition_key}")
    
    # 元データを読み込み
    df = pd.read_csv(sales_config.raw_data_path)
    df["date"] = pd.to_datetime(df["date"])
    
    # パーティションキーに該当する日付のデータのみ抽出
    target_date = pd.to_datetime(partition_key)
    df_filtered = df[df["date"].dt.date == target_date.date()]
    
    # 売上金額を計算
    df_filtered = df_filtered.copy()
    df_filtered["amount"] = df_filtered["quantity"] * df_filtered["price"]
    
    # 階層化したパスに出力（べき等性を担保）
    output_path = partition_key_to_path(
        partition_key,
        sales_config.output_dir,
        "sales.csv"
    )
    
    # ディレクトリがなければ作成
    os.makedirs(os.path.dirname(output_path), exist_ok=True)
    
    # CSVとして保存
    df_filtered.to_csv(output_path, index=False)
    
    return MaterializeResult(
        metadata={
            "partition_key": partition_key,
            "row_count": len(df_filtered),
            "total_amount": int(df_filtered["amount"].sum()) if len(df_filtered) > 0 else 0,
            "output_path": MetadataValue.path(output_path),
        }
    )
```

ポイントは以下の3つです。

1. **`partitions_def=daily_partitions`**: Assetにパーティション定義を紐付け
2. **`context.partition_key`**: 実行中のパーティションキーを取得
3. **`partition_key_to_path()`**: パーティションキーを階層化パスに変換

:::message
**💡 制御の逆転のメリット**

パーティション化されたAssetを実行すると、Dagsterが自動的に「今はどの日の処理か」を`context`を通じて教えてくれます。開発者がループ処理を書いて日付を回す必要がなく、「どの日を処理するか」の制御はDagster側に委ねられます。

これにより、コードは「1日分のデータをどう処理するか」に集中でき、シンプルで保守しやすくなります。
:::

### 階層化したファイルパス

パーティションキーをそのままファイル名にするのではなく、階層化したディレクトリ構造に変換しています。

```
/opt/dagster/app/data/
├── 2026/
│   └── 01/
│       ├── 01/
│       │   └── sales.csv
│       ├── 02/
│       │   └── sales.csv
│       └── ...
```

この構造により、特定の年月のデータを探しやすく、S3などのオブジェクトストレージとも相性が良くなります。

### Definitionsへの登録

パーティション対応Assetを`Definitions`に追加します。

```python
from dagster import Definitions

from .assets import raw_sales_data, cleaned_sales_data, sales_summary
from .partitioned_assets import partitioned_sales_data, partitioned_sales_summary
from .resources import SalesDataConfig
from .schedules import daily_sales_schedule

defs = Definitions(
    assets=[
        # 非パーティションAsset
        raw_sales_data,
        cleaned_sales_data,
        sales_summary,
        # パーティションAsset
        partitioned_sales_data,
        partitioned_sales_summary,
    ],
    resources={
        "sales_config": SalesDataConfig(),
    },
    schedules=[daily_sales_schedule],
)
```

## UIでの操作

### パーティション一覧

Catalogで`partitioned_sales_data`を開くと、「Partitions」タブでパーティション一覧が表示されます。

![パーティション一覧](/images/dagster-partition/partition-list.png)

`start_date`から現在日までのパーティションが自動生成され、それぞれの状態（Materialized / Missing）が確認できます。

### 単一パーティションのMaterialize

特定のパーティションを選択して「Materialize」を実行すると、そのパーティションのみが処理されます。

![パーティション詳細](/images/dagster-partition/partition-detail.png)

メタデータには以下が記録されます。

| キー | 値 |
|------|-----|
| partition_key | 2026-01-01 |
| row_count | 1 |
| total_amount | 1000 |
| output_path | /opt/dagster/app/data/2026/01/01/sales.csv |

### バックフィル

複数のパーティションを一括で処理する「バックフィル」も実行できます。

「Materialize」ボタンをクリックし、タイムラインをドラッグして範囲を選択します。

![バックフィルダイアログ](/images/dagster-partition/backfill-dialog.png)

「Launch backfill」をクリックすると、選択した範囲のパーティションが順次実行されます。

:::message
バックフィルでは、既にMaterializeされたパーティションも再実行されます。べき等性が担保されているため、同じ結果が上書きされるだけで問題ありません。

「Backfill only failed and missing partitions within selection」にチェックを入れると、未実行・失敗のパーティションのみを対象にできます。
:::

:::message
**💡 バックフィルの実行効率（並列実行）**

Docker Compose環境ではリソースが限られますが、本番環境（Kubernetes等）では、バックフィル実行時に複数のパーティションを並列で同時実行することも可能です。これにより、1年分の過去データを数分で作り直すといった処理も実現できます。
:::

### Partition Health画面

バックフィル完了後、Partition Health画面で処理状況を一目で確認できます。

![Partition Health](/images/dagster-partition/partition-health.png)

| 状態 | 色 | 説明 |
|------|-----|------|
| Materialized | 緑 | 処理済み |
| Missing | グレー | 未処理 |
| Failed | 赤 | 失敗 |

緑のマス目が並ぶこの画面は、データの「健康診断」が一目でできるDagsterの大きな魅力です。どの期間のデータが揃っているか、どこに欠損があるかが視覚的にわかり、再処理すべきパーティションを即座に判断できます。

## PartitionとScheduleの連携

パーティション対応Assetを定期実行するには、`build_schedule_from_partitioned_job`を使う方法もありますが、シンプルに`ScheduleDefinition`で最新パーティションを対象にすることも可能です。

```python
from dagster import (
    ScheduleDefinition,
    AssetSelection,
    DefaultScheduleStatus,
    build_schedule_from_partitioned_job,
    define_asset_job,
)
from .partitions import daily_partitions

# パーティション対応Assetのジョブを定義
partitioned_job = define_asset_job(
    name="partitioned_sales_job",
    selection=AssetSelection.assets(partitioned_sales_data, partitioned_sales_summary),
    partitions_def=daily_partitions,
)

# 日次で最新パーティションを実行するSchedule
partitioned_schedule = build_schedule_from_partitioned_job(
    job=partitioned_job,
    default_status=DefaultScheduleStatus.STOPPED,
)
```

これにより、毎日その日のパーティションが自動的にMaterializeされます。

## まとめ

本記事では、Partitionを使ったデータの分割管理について検証しました。

**検証した内容**
- DailyPartitionsDefinitionで日付パーティションを定義
- context.partition_keyでパーティションキーを取得
- 階層化したファイルパス（/2026/01/15/sales.csv）への出力
- 単一パーティションのMaterialize
- バックフィルによる一括処理
- Partition Health画面での状態可視化

**べき等性のポイント**
- 同じパーティションキーは常に同じ出力先
- 何度実行しても同じ結果
- 失敗時の再実行が安全

Partitionを使うことで、大量のデータを効率的に処理し、問題が発生した場合も影響範囲を最小限に抑えて再処理できるようになりました。

## 参考リンク

- [Dagster公式ドキュメント - Partitions](https://docs.dagster.io/concepts/partitions-schedules-sensors/partitions)
- [Dagster公式ドキュメント - Backfills](https://docs.dagster.io/concepts/partitions-schedules-sensors/backfills)
- [検証用リポジトリ](https://github.com/toshiro3/workflow-orchestration-lab/tree/v3-partition)
