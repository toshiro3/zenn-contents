---
title: "Dagster入門 - Docker Composeで始めるデータパイプライン"
emoji: "🔷"
type: "tech"
topics: ["dagster", "docker", "python", "データエンジニアリング"]
published: false
---

## はじめに

データパイプラインの構築・運用において、ワークフローオーケストレーションツールは重要な役割を果たしています。Apache Airflowが長らくデファクトスタンダードとして使われてきましたが、近年はDagsterやPrefectといった新しいツールも注目を集めているようです。

本記事では、Dagsterをローカル環境で動かしながら、基本的な使い方を検証していきます。

**本記事で扱う内容**
- Docker Composeによる環境構築
- Assetの定義と実行
- 複数Assetの依存関係
- CSVファイルの読み込み
- メタデータの出力とUIでの確認

検証用のコードはGitHubで公開しています。
https://github.com/toshiro3/workflow-orchestration-lab

## Dagsterとは

Dagsterは、データパイプラインの開発・運用・監視を行うためのオーケストレーションツールです。Dagster Labs社（旧Elementl社）によって開発され、オープンソースとして公開されています。

### 特徴

公式ドキュメントや各種記事によると、Dagsterには以下のような特徴があるようです。

**Asset-centric（アセット中心）のアプローチ**

従来のワークフローツールがタスク（処理）を中心に設計されているのに対し、DagsterはAsset（データ資産）を中心に設計されています。「何を作るか」を定義することで、「どう作るか」は自動的に決まるという考え方のようです。

**Software-defined Assets**

Assetをコードで定義することで、データの依存関係、スケジュール、テストなどを一元管理できます。これにより、データパイプラインの品質と保守性が向上するとされています。

**豊富なUI**

Dagsterは強力なWeb UIを提供しており、Assetの依存関係（Lineage）、実行履歴、メタデータなどを視覚的に確認できます。

## 環境構築

Docker Composeを使用して、ローカル環境にDagsterをセットアップしました。

### アーキテクチャ

Dagsterの本番環境は、複数のコンポーネントで構成されます。

| コンポーネント | 役割 |
|--------------|------|
| PostgreSQL | メタデータストレージ（実行履歴、スケジュールなど） |
| User Code | パイプラインコードを実行するgRPCサーバー |
| Webserver | Dagster UI |
| Daemon | スケジューラー・センサーの実行 |

User CodeがgRPCサーバーとして分離されている点は、Dagsterの重要なアーキテクチャ上の特徴です。WebserverやDaemonとパイプラインの実行環境が分離されているため、Pythonのバージョンやライブラリの依存関係が異なる複数のUser Codeコンテナを立てて共存させることができます。

```
┌─────────────┐    ┌─────────────┐
│  Webserver  │    │   Daemon    │
│  (UI)       │    │ (Scheduler) │
└──────┬──────┘    └──────┬──────┘
       │                  │
       └────────┬─────────┘
                │
       ┌────────▼────────┐
       │    User Code    │
       │  (gRPC Server)  │
       └────────┬────────┘
                │
       ┌────────▼────────┐
       │   PostgreSQL    │
       │  (Metadata)     │
       └─────────────────┘
```

### ディレクトリ構成

以下のようなディレクトリ構成で環境を作成します。

```
dagster/
├── docker-compose.yml
├── dagster/
│   ├── Dockerfile
│   ├── dagster.yaml
│   └── workspace.yaml
└── user_code/
    ├── Dockerfile
    ├── requirements.txt
    ├── data/
    │   └── sales.csv
    └── my_dagster_project/
        ├── __init__.py
        └── assets.py
```

### docker-compose.yml

```yaml
services:
  # PostgreSQL - メタデータストレージ
  postgresql:
    image: postgres:17
    container_name: dagster_postgresql
    environment:
      POSTGRES_USER: dagster
      POSTGRES_PASSWORD: dagster
      POSTGRES_DB: dagster
    volumes:
      - dagster_postgres_data:/var/lib/postgresql/data
    networks:
      - dagster_network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dagster -d dagster"]
      interval: 5s
      timeout: 5s
      retries: 5

  # User Code - パイプラインコードを実行するgRPCサーバー
  user_code:
    build:
      context: ./user_code
      dockerfile: Dockerfile
    container_name: dagster_user_code
    image: dagster_user_code_image
    pull_policy: build
    restart: always
    environment:
      DAGSTER_POSTGRES_USER: dagster
      DAGSTER_POSTGRES_PASSWORD: dagster
      DAGSTER_POSTGRES_DB: dagster
      DAGSTER_POSTGRES_HOST: postgresql
      DAGSTER_CURRENT_IMAGE: dagster_user_code_image
    networks:
      - dagster_network
    depends_on:
      postgresql:
        condition: service_healthy

  # Webserver - Dagster UI
  webserver:
    build:
      context: ./dagster
      dockerfile: Dockerfile
    container_name: dagster_webserver
    entrypoint:
      - dagster-webserver
      - -h
      - "0.0.0.0"
      - -p
      - "3000"
      - -w
      - workspace.yaml
    ports:
      - "3000:3000"
    environment:
      DAGSTER_POSTGRES_USER: dagster
      DAGSTER_POSTGRES_PASSWORD: dagster
      DAGSTER_POSTGRES_DB: dagster
      DAGSTER_POSTGRES_HOST: postgresql
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /tmp/io_manager_storage:/tmp/io_manager_storage
    networks:
      - dagster_network
    depends_on:
      postgresql:
        condition: service_healthy
      user_code:
        condition: service_started

  # Daemon - スケジューラー・センサー
  daemon:
    build:
      context: ./dagster
      dockerfile: Dockerfile
    container_name: dagster_daemon
    entrypoint:
      - dagster-daemon
      - run
    restart: on-failure
    environment:
      DAGSTER_POSTGRES_USER: dagster
      DAGSTER_POSTGRES_PASSWORD: dagster
      DAGSTER_POSTGRES_DB: dagster
      DAGSTER_POSTGRES_HOST: postgresql
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /tmp/io_manager_storage:/tmp/io_manager_storage
    networks:
      - dagster_network
    depends_on:
      postgresql:
        condition: service_healthy
      user_code:
        condition: service_started

networks:
  dagster_network:
    driver: bridge

volumes:
  dagster_postgres_data:
```

### Dagster設定ファイル

**dagster/Dockerfile**

```dockerfile
FROM python:3.11-slim

RUN pip install --no-cache-dir \
    dagster \
    dagster-webserver \
    dagster-postgres \
    dagster-docker

ENV DAGSTER_HOME=/opt/dagster/dagster_home

RUN mkdir -p $DAGSTER_HOME

COPY dagster.yaml workspace.yaml $DAGSTER_HOME/

WORKDIR $DAGSTER_HOME
```

**dagster/dagster.yaml**

Dagsterインスタンスの設定ファイルです。ストレージ、スケジューラー、実行環境などを定義します。

今回は`run_launcher`に`DockerRunLauncher`を指定しています。これは各Jobの実行ごとに新しいDockerコンテナを立ち上げる設定で、本番運用を見据えた構成です。ローカル検証で軽量に動かしたい場合は、`DefaultRunLauncher`（現在のプロセス内で実行）を使う選択肢もあります。

:::message
`network: dagster_dagster_network`の部分は、Docker Composeが自動生成するネットワーク名（`プロジェクト名_ネットワーク名`の形式）に依存します。ご自身の環境でエラーが発生した場合は、`docker network ls`コマンドで実際のネットワーク名を確認し、適宜修正してください。
:::

```yaml
scheduler:
  module: dagster.core.scheduler
  class: DagsterDaemonScheduler

run_coordinator:
  module: dagster.core.run_coordinator
  class: QueuedRunCoordinator

run_launcher:
  module: dagster_docker
  class: DockerRunLauncher
  config:
    env_vars:
      - DAGSTER_POSTGRES_USER
      - DAGSTER_POSTGRES_PASSWORD
      - DAGSTER_POSTGRES_DB
      - DAGSTER_POSTGRES_HOST
    network: dagster_dagster_network
    container_kwargs:
      volumes:
        - /tmp/io_manager_storage:/tmp/io_manager_storage

run_storage:
  module: dagster_postgres.run_storage
  class: PostgresRunStorage
  config:
    postgres_db:
      hostname:
        env: DAGSTER_POSTGRES_HOST
      username:
        env: DAGSTER_POSTGRES_USER
      password:
        env: DAGSTER_POSTGRES_PASSWORD
      db_name:
        env: DAGSTER_POSTGRES_DB
      port: 5432

schedule_storage:
  module: dagster_postgres.schedule_storage
  class: PostgresScheduleStorage
  config:
    postgres_db:
      hostname:
        env: DAGSTER_POSTGRES_HOST
      username:
        env: DAGSTER_POSTGRES_USER
      password:
        env: DAGSTER_POSTGRES_PASSWORD
      db_name:
        env: DAGSTER_POSTGRES_DB
      port: 5432

event_log_storage:
  module: dagster_postgres.event_log
  class: PostgresEventLogStorage
  config:
    postgres_db:
      hostname:
        env: DAGSTER_POSTGRES_HOST
      username:
        env: DAGSTER_POSTGRES_USER
      password:
        env: DAGSTER_POSTGRES_PASSWORD
      db_name:
        env: DAGSTER_POSTGRES_DB
      port: 5432
```

**dagster/workspace.yaml**

コードロケーション（ユーザーコードの場所）を定義します。

```yaml
load_from:
  - grpc_server:
      host: user_code
      port: 4000
      location_name: "my_dagster_project"
```

### User Code設定ファイル

**user_code/Dockerfile**

```dockerfile
FROM python:3.11-slim

WORKDIR /opt/dagster/app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY my_dagster_project/ ./my_dagster_project/
COPY data/ ./data/

EXPOSE 4000

CMD ["dagster", "api", "grpc", "-h", "0.0.0.0", "-p", "4000", "-m", "my_dagster_project"]
```

**user_code/requirements.txt**

```
dagster
dagster-postgres
dagster-docker
pandas
tabulate
```

### 起動

```bash
# ビルド＆起動
docker compose up -d --build

# 起動確認
docker compose ps
```

```
NAME                 IMAGE                     COMMAND                  SERVICE      CREATED          STATUS                    PORTS
dagster_daemon       dagster-daemon            "dagster-daemon run"     daemon       13 seconds ago   Up 6 seconds
dagster_postgresql   postgres:17               "docker-entrypoint.s…"   postgresql   14 seconds ago   Up 12 seconds (healthy)   5432/tcp
dagster_user_code    dagster_user_code_image   "dagster api grpc -h…"   user_code    13 seconds ago   Up 6 seconds              4000/tcp
dagster_webserver    dagster-webserver         "dagster-webserver -…"   webserver    13 seconds ago   Up 6 seconds              0.0.0.0:3000->3000/tcp
```

4つのコンテナが起動したら、http://localhost:3000 でDagster UIにアクセスできます。

## 最初のAssetを作る

環境が準備できたので、実際にAssetを定義してみました。

### Assetとは

DagsterにおけるAssetは、パイプラインによって生成されるデータを表します。例えば、以下のようなものがAssetになります。

- データベースのテーブル
- CSVやParquetファイル
- 機械学習モデル
- レポート

Assetを定義することで、「何を作るか」と「どう作るか」を同時に宣言できます。

### シンプルなAsset定義

**user_code/my_dagster_project/assets.py**

```python
import pandas as pd
from dagster import asset, AssetExecutionContext, MaterializeResult, MetadataValue


@asset(
    description="CSVファイルから売上データを読み込む",
)
def raw_sales_data(context: AssetExecutionContext) -> MaterializeResult:
    """CSVファイルから売上データを読み込むAsset"""
    df = pd.read_csv("/opt/dagster/app/data/sales.csv")
    
    context.log.info(f"Loaded {len(df)} rows from sales.csv")
    
    return MaterializeResult(
        metadata={
            "row_count": len(df),
            "columns": MetadataValue.text(", ".join(df.columns.tolist())),
            "preview": MetadataValue.md(df.head().to_markdown()),
        },
    )
```

`@asset`デコレータを使用して関数をAssetとして定義します。主なポイントは以下の通りです。

- **`description`**: Assetの説明。UIに表示されます
- **`context`**: 実行コンテキスト。ログ出力などに使用
- **`MaterializeResult`**: メタデータを含む実行結果を返す

### Definitionsへの登録

定義したAssetは、`Definitions`に登録する必要があります。

`user_code/my_dagster_project/__init__.py`

```python
from dagster import Definitions

from .assets import raw_sales_data, cleaned_sales_data, sales_summary

defs = Definitions(
    assets=[raw_sales_data, cleaned_sales_data, sales_summary],
)
```

## 複数Assetの依存関係

実際のデータパイプラインでは、複数のAssetが依存関係を持って連携します。今回は3つのAssetで依存関係を構築してみました。

### 依存関係の定義

Dagsterでは、Assetの依存関係を定義する方法が2つあります。

**方法1: 関数引数による定義（推奨）**

上流のAsset名を関数の引数として受け取る方法です。これがDagsterの標準的なやり方で、I/Oマネージャーを介したデータの受け渡しが可能になります。

この方式では、Dagsterが裏側で自動的にデータをPickleやParquet形式で保存し、次のAssetでロードしてくれます。開発者はデータの保存・読み込みを意識する必要がなく、処理ロジックに集中できます。これが単なるタスク管理ツールとの大きな違いであり、「データプラットフォーム」としてのDagsterの強みです。

```python
@asset
def raw_data() -> pd.DataFrame:
    return pd.read_csv("data.csv")

@asset
def cleaned_data(raw_data: pd.DataFrame) -> pd.DataFrame:  # 引数名がAsset名と一致
    df = raw_data.copy()
    df["amount"] = df["quantity"] * df["price"]
    return df
```

**方法2: depsパラメータによる定義**

`deps`パラメータで依存関係を明示的に指定する方法です。ファイルの読み書きなど、副作用を伴う処理で使用します。

```python
@asset(
    deps=["raw_sales_data"],  # 依存関係を明示的に指定
)
def cleaned_sales_data(context: AssetExecutionContext) -> MaterializeResult:
    df = pd.read_csv("/opt/dagster/app/data/sales.csv")  # 自分でファイルを読み込む
    ...
```

今回の検証では、ファイルの読み書きを明示的に行う`deps`を使用しました。実際の運用では、I/Oマネージャーと引数による依存定義を組み合わせることで、Dagsterにデータの受け渡しを任せることも可能です。

以下が今回のコードです。

```python
@asset(
    description="売上金額を計算してデータをクレンジング",
    deps=["raw_sales_data"],  # 依存関係を明示的に指定
)
def cleaned_sales_data(context: AssetExecutionContext) -> MaterializeResult:
    """売上金額を計算し、データをクレンジングするAsset"""
    df = pd.read_csv("/opt/dagster/app/data/sales.csv")
    
    df["amount"] = df["quantity"] * df["price"]
    df["date"] = pd.to_datetime(df["date"])
    
    total_amount = df["amount"].sum()
    context.log.info(f"Cleaned data: {len(df)} rows, total amount: {total_amount}")
    
    # 中間結果をCSVとして保存
    output_path = "/opt/dagster/app/data/cleaned_sales.csv"
    df.to_csv(output_path, index=False)
    
    return MaterializeResult(
        metadata={
            "row_count": len(df),
            "total_amount": int(total_amount),
            "date_range": MetadataValue.text(
                f"{df['date'].min().strftime('%Y-%m-%d')} ~ {df['date'].max().strftime('%Y-%m-%d')}"
            ),
            "output_path": MetadataValue.path(output_path),
            "preview": MetadataValue.md(df.head().to_markdown()),
        }
    )


@asset(
    description="商品別の集計統計を計算",
    deps=["cleaned_sales_data"],
)
def sales_summary(context: AssetExecutionContext) -> MaterializeResult:
    """商品別の集計統計を計算するAsset"""
    df = pd.read_csv("/opt/dagster/app/data/cleaned_sales.csv")
    
    summary = df.groupby("product").agg({
        "quantity": "sum",
        "amount": "sum",
    }).reset_index()
    
    summary.columns = ["product", "total_quantity", "total_amount"]
    summary = summary.sort_values("total_amount", ascending=False)
    
    # 結果をCSVとして保存
    output_path = "/opt/dagster/app/data/sales_summary.csv"
    summary.to_csv(output_path, index=False)
    
    context.log.info(f"Summary stats:\n{summary.to_string()}")
    
    return MaterializeResult(
        metadata={
            "product_count": len(summary),
            "top_product": summary.iloc[0]["product"],
            "top_product_amount": int(summary.iloc[0]["total_amount"]),
            "output_path": MetadataValue.path(output_path),
            "summary_table": MetadataValue.md(summary.to_markdown(index=False)),
        }
    )
```

これにより、以下のような依存関係が構築されます。

```
raw_sales_data → cleaned_sales_data → sales_summary
```

### Lineageグラフでの確認

Dagster UIの「Lineage」メニューから、Assetの依存関係をグラフで確認できます。

各Assetの状態（未実行、実行中、成功、失敗）もリアルタイムで表示されます。

![Lineageグラフ](/images/dagster-introduction/lineage-graph.png)
*Dagster UIで表示されるLineageグラフ*

## メタデータの出力

Dagsterの強力な機能の1つが、Assetにメタデータを付与できることです。メタデータはUIで確認でき、データの品質管理やデバッグに役立ちそうです。

### MetadataValueの種類

| 種類 | 用途 | 例 |
|-----|------|-----|
| `int` / `float` | 数値 | 行数、合計値 |
| `MetadataValue.text()` | テキスト | カラム名、日付範囲 |
| `MetadataValue.md()` | Markdown | テーブルのプレビュー |
| `MetadataValue.path()` | ファイルパス | 出力先パス |
| `MetadataValue.url()` | URL | 外部リンク |
| `MetadataValue.json()` | JSON | 構造化データ |

※ `MetadataValue.md()`で`df.to_markdown()`を使用する場合は、`tabulate`ライブラリが必要です。

### UIでのメタデータ確認

Assetの詳細画面では、Materialize時に出力したメタデータを確認できました。

例えば、`sales_summary`のメタデータでは以下の情報が表示されていました。

- `product_count`: 3（商品数）
- `top_product`: B（売上トップの商品）
- `top_product_amount`: 26,000（トップ商品の売上）
- `summary_table`: 商品別集計テーブル

このようにメタデータを活用することで、パイプラインの実行結果を詳細に把握できそうです。

## 基本概念の整理

ここで、Dagsterの主要な概念を整理しておきます。

### Asset

パイプラインによって生成されるデータ資産。テーブル、ファイル、モデルなど。

```python
@asset
def my_asset():
    return data
```

### Op（Operation）

単一の処理単位。Assetを構成する基本要素です。`@asset`は内部的に`@op`をラップしています。

```python
@op
def my_op():
    # 処理
    pass
```

### Job

Opの集合を定義し、実行可能な単位にまとめたもの。

```python
@job
def my_job():
    op1()
    op2()
```

※ Dagster 1.0以降では、Assetが主役となっており、Op/Jobを直接書く機会は減っているようです。Op/Jobは、Assetに馴染まない非データ生成的な処理（通知など）や、レガシーな記述で使われることが多いとされています。

### Resource

データベース接続やAPIクライアントなど、外部リソースへのアクセスを抽象化したもの。

```python
@resource
def my_database():
    return DatabaseConnection()
```

### Schedule

Jobの定期実行を定義。

```python
@schedule(cron_schedule="0 0 * * *", job=my_job)
def daily_schedule():
    return {}
```

### Sensor

イベント駆動でJobを実行するトリガー。

```python
@sensor(job=my_job)
def my_sensor():
    if new_file_exists():
        yield RunRequest()
```

## まとめ

本記事では、Dagsterの環境構築からAssetの定義・実行までを検証しました。

**検証した内容**
- Docker Composeによる4コンテナ構成の環境構築
- `@asset`デコレータによるAsset定義
- `deps`による依存関係の定義
- `MaterializeResult`と`MetadataValue`によるメタデータ出力
- Dagster UIでのLineage、ログ、メタデータの確認

Dagsterの特徴であるAsset-centricなアプローチにより、「何を作るか」を中心にパイプラインを設計できることが体感できたように思えます。

**次回予告**

次回は、以下のような内容を予定しています。

- Resourceを使った設定の外部化
- Scheduleによる定期実行
- Partitionによるデータ分割

## 参考リンク

- [Dagster公式ドキュメント](https://docs.dagster.io/)
- [Dagster GitHub](https://github.com/dagster-io/dagster)
- [検証用リポジトリ](https://github.com/toshiro3/workflow-orchestration-lab)
