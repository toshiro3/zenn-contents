---
title: "Dagster入門② - ResourceとScheduleで実用的なパイプラインへ"
emoji: "🔷"
type: "tech"
topics: ["dagster", "docker", "python", "データエンジニアリング"]
published: false
---

## はじめに

[前回の記事](https://zenn.dev/toshiro3/articles/dagster-introduction)では、Docker ComposeでDagster環境を構築し、Assetの定義と依存関係について検証しました。

今回は、より実用的なパイプラインに近づけるため、以下の機能を検証していきます。

**本記事で扱う内容**
- Resourceを使った設定の外部化
- Launchpadからの実行時パラメータ変更
- Scheduleによる定期実行

検証用のコードはGitHubで公開しています。
https://github.com/toshiro3/workflow-orchestration-lab/tree/v2-resource-schedule

## Resourceとは

Resourceは、パイプラインで使用する設定や外部リソースへの接続を管理する仕組みです。データベース接続、APIクライアント、ファイルパスなどを抽象化し、コードから分離できます。

### なぜResourceを使うのか

前回のコードでは、ファイルパスがハードコードされていました。

```python
# ハードコードされたパス
df = pd.read_csv("/opt/dagster/app/data/sales.csv")
output_path = "/opt/dagster/app/data/cleaned_sales.csv"
```

この書き方には以下の問題があります。

- 環境ごと（開発/本番）にパスを変えられない
- テスト時にモック化しにくい
- パスを変更するたびにコードを修正する必要がある

Resourceを使うことで、これらの問題を解決できます。

### ConfigurableResource

Dagsterでは、`ConfigurableResource`クラスを使ってResourceを定義します。これはPydanticをベースにしており、型安全な設定管理が可能です。

```python
from dagster import ConfigurableResource


class SalesDataConfig(ConfigurableResource):
    """売上データパイプラインの設定を管理するResource"""
    
    input_dir: str = "/opt/dagster/app/data"
    output_dir: str = "/opt/dagster/app/data"
    
    @property
    def raw_data_path(self) -> str:
        return f"{self.input_dir}/sales.csv"
    
    @property
    def cleaned_data_path(self) -> str:
        return f"{self.output_dir}/cleaned_sales.csv"
    
    @property
    def summary_data_path(self) -> str:
        return f"{self.output_dir}/sales_summary.csv"
```

Pydanticベースなので、VS Codeなどのエディタで型補完が効きます。これにより、開発時のタイプミスを防ぎ、開発体験（DX）が向上します。

## Resourceの実装

### ファイル構成

前回からファイル構成を変更し、`resources.py`と`schedules.py`を追加しました。

```
my_dagster_project/
├── __init__.py      # Definitions
├── assets.py        # Asset定義
├── resources.py     # Resource定義（新規）
└── schedules.py     # Schedule定義（新規）
```

### Resourceの定義

`resources.py`を作成し、`ConfigurableResource`を定義します。

```python
from dagster import ConfigurableResource


class SalesDataConfig(ConfigurableResource):
    """売上データパイプラインの設定を管理するResource
    
    Pydanticベースなので、エディタで型補完が効きます。
    環境ごとの設定切り替え（開発/本番）に適しています。
    """
    
    # 入力データのベースディレクトリ
    input_dir: str = "/opt/dagster/app/data"
    
    # 出力データのベースディレクトリ
    output_dir: str = "/opt/dagster/app/data"
    
    @property
    def raw_data_path(self) -> str:
        """入力CSVファイルのパス"""
        return f"{self.input_dir}/sales.csv"
    
    @property
    def cleaned_data_path(self) -> str:
        """クレンジング後のCSVファイルのパス"""
        return f"{self.output_dir}/cleaned_sales.csv"
    
    @property
    def summary_data_path(self) -> str:
        """集計結果のCSVファイルのパス"""
        return f"{self.output_dir}/sales_summary.csv"
```

### AssetでのResource利用

Assetの引数にResourceを追加することで、設定を注入できます。引数名は`Definitions`に登録したキー名と一致させる必要があります。

```python
from dagster import asset, AssetExecutionContext, MaterializeResult, MetadataValue
from .resources import SalesDataConfig


@asset(description="CSVファイルから売上データを読み込む")
def raw_sales_data(
    context: AssetExecutionContext,
    sales_config: SalesDataConfig,  # 引数名はDefinitionsのキー名と一致させる
) -> MaterializeResult:
    # Resourceから設定を取得
    df = pd.read_csv(sales_config.raw_data_path)
    
    context.log.info(f"Loaded {len(df)} rows from {sales_config.raw_data_path}")
    
    return MaterializeResult(
        metadata={
            "row_count": len(df),
            "input_path": MetadataValue.path(sales_config.raw_data_path),
            # ...
        },
    )
```

ハードコードされたパスが`sales_config.raw_data_path`に置き換わりました。

:::message
**💡 Tips: 引数名による使い分けのルール**

- `sales_config`（リソース名と一致）: Launchpadの`resources`セクション（全体共通設定）と連動
- `config`（予約語）: Launchpadの`ops`セクション（そのAsset個別の設定）と連動

今回のように全体で共有したいパスなどは、リソース名と一致させた引数名で受け取るのがスマートです。`config`という名前を使うと`ops`セクション側と連動してしまうため、意図しない動作になる可能性があります。
:::

### Definitionsへの登録

Resourceは`Definitions`に登録する必要があります。

```python
from dagster import Definitions
from .assets import raw_sales_data, cleaned_sales_data, sales_summary
from .resources import SalesDataConfig
from .schedules import daily_sales_schedule

defs = Definitions(
    assets=[raw_sales_data, cleaned_sales_data, sales_summary],
    resources={
        "sales_config": SalesDataConfig(),  # Resourceを登録
    },
    schedules=[daily_sales_schedule],
)
```

## LaunchpadでUIからパラメータを変更

ConfigurableResourceの便利な点は、Launchpadから実行時に設定を変更できることです。

### Launchpadを開く

Asset詳細画面で、「Materialize」ボタンの横にあるドロップダウンから「Open launchpad」を選択します。

![Launchpadを開く](/images/dagster-resource-schedule/open-launchpad.png)

### パラメータを変更

Launchpadが開くと、Resourceの設定がYAML形式で表示されます。

![Launchpad画面](/images/dagster-resource-schedule/launchpad.png)

`resources`セクションの値を変更することで、実行時にパラメータを上書きできます。

```yaml
resources:
  sales_config:
    config:
      input_dir: /opt/dagster/app/data
      output_dir: /tmp  # 出力先を変更
```

### 実行結果の確認

`output_dir`を`/tmp`に変更してMaterializeを実行すると、出力先が変更されていることがログで確認できます。

![実行結果](/images/dagster-resource-schedule/run-result.png)

`output_path`が`/tmp/cleaned_sales.csv`になっていることが確認できました。

このように、コードを変更せずにパラメータを調整できるのがResourceの利点です。

## Scheduleとは

Scheduleは、Assetを定期的にMaterializeするための仕組みです。cron式でスケジュールを指定し、指定した時刻に自動で実行されます。

### Daemonの役割

Scheduleの実行には、**Daemon**コンテナが必要です。前回の環境構築で立てた4つのコンテナのうち、Daemonがスケジューラーとして動作します。

| コンポーネント | 役割 |
|--------------|------|
| PostgreSQL | メタデータストレージ |
| User Code | パイプラインコード |
| Webserver | Dagster UI |
| **Daemon** | **スケジューラー・センサーの実行** |

:::message
Daemonが停止していると、Scheduleは実行されません。`docker compose ps`でDaemonが起動していることを確認してください。
:::

## Scheduleの実装

### Schedule定義

`schedules.py`を作成し、Scheduleを定義します。

```python
from dagster import ScheduleDefinition, AssetSelection, DefaultScheduleStatus


daily_sales_schedule = ScheduleDefinition(
    name="daily_sales_schedule",
    target=AssetSelection.all(),  # すべてのAssetを対象
    cron_schedule="0 9 * * *",    # 毎日9:00 UTCに実行
    default_status=DefaultScheduleStatus.STOPPED,  # 初期状態は停止
)
```

| パラメータ | 説明 |
|-----------|------|
| `name` | Scheduleの名前 |
| `target` | 対象のAsset（`AssetSelection.all()`で全Asset） |
| `cron_schedule` | cron式（`0 9 * * *`は毎日9:00） |
| `default_status` | 初期状態（`STOPPED`または`RUNNING`） |

### cron式の書き方

```
┌───────────── 分 (0-59)
│ ┌───────────── 時 (0-23)
│ │ ┌───────────── 日 (1-31)
│ │ │ ┌───────────── 月 (1-12)
│ │ │ │ ┌───────────── 曜日 (0-6, 0=日曜)
│ │ │ │ │
* * * * *
```

よく使うパターン：

| cron式 | 意味 |
|--------|------|
| `0 9 * * *` | 毎日9:00 |
| `0 0 * * *` | 毎日0:00（深夜） |
| `0 */6 * * *` | 6時間ごと |
| `0 9 * * 1` | 毎週月曜9:00 |
| `0 9 1 * *` | 毎月1日9:00 |

### タイムゾーンの設定

デフォルトではUTCで動作します。日本時間（JST）で指定したい場合は、`execution_timezone`を追加してください。

```python
daily_sales_schedule = ScheduleDefinition(
    name="daily_sales_schedule",
    target=AssetSelection.all(),
    cron_schedule="0 9 * * *",  # 日本時間の9:00に実行
    execution_timezone="Asia/Tokyo",  # タイムゾーンを指定
    default_status=DefaultScheduleStatus.STOPPED,
)
```

### Definitionsへの登録

Scheduleも`Definitions`に登録します（前述のコード参照）。

## UIでのSchedule操作

### Automationメニュー

左メニューの「Automation」を開くと、登録されたScheduleが一覧表示されます。

![Automation一覧](/images/dagster-resource-schedule/automation-list.png)

### Schedule詳細

Scheduleをクリックすると詳細画面が開きます。

![Schedule詳細](/images/dagster-resource-schedule/schedule-detail.png)

| 項目 | 説明 |
|------|------|
| Latest tick | 最後に実行された時刻 |
| Next tick | 次回実行予定時刻 |
| Target | 対象のAsset |
| Running | ON/OFF状態 |
| Schedule | cron式 |
| Execution timezone | タイムゾーン（デフォルトUTC） |

### ScheduleのON/OFF

トグルスイッチでScheduleを有効/無効にできます。ONにすると、次回実行予定時刻（Next tick）が表示されます。

## まとめ

本記事では、ResourceとScheduleを使って、より実用的なパイプラインに近づけることができました。

**検証した内容**
- ConfigurableResourceで設定を外部化
- Pydanticベースの型安全な設定管理
- LaunchpadからUIで実行時パラメータを変更
- ScheduleDefinitionで定期実行を設定
- UIからのSchedule ON/OFF操作
- Daemonがスケジュール実行に必要なことを確認

Resourceを使うことで、環境ごとの設定切り替えやテストが容易になり、Scheduleを使うことで定期的なデータ更新を自動化できるようになりました。

**次回予告**

次回は、Partitionを使ったデータの分割管理について検証する予定です。

- 日付パーティションでのデータ処理
- べき等性（Idempotency）を意識した設計
- バックフィルによる再処理
- Partition Health画面での欠損確認

## 参考リンク

- [Dagster公式ドキュメント - Resources](https://docs.dagster.io/concepts/resources)
- [Dagster公式ドキュメント - Schedules](https://docs.dagster.io/concepts/automation/schedules)
- [検証用リポジトリ](https://github.com/toshiro3/workflow-orchestration-lab/tree/v2-resource-schedule)
