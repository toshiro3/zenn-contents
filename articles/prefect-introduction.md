---
title: "Prefect 3.0入門 - Docker Composeでローカル環境を構築してDagster・Airflowと比較"
emoji: "🔄"
type: "tech"
topics: ["prefect", "python", "docker", "workflow", "dataengineering"]
published: false
---

# Prefect 3.0入門 - Docker Composeでローカル環境を構築してDagster・Airflowと比較

## はじめに

ワークフローオーケストレーションツールの比較検証として、これまでDagsterとAirflow 3.xの記事を書いてきました。今回は3つ目のツールとして**Prefect 3.0**を検証します。

同じCSVパイプライン（読み込み → クレンジング → 集計）を実装し、Dagster・Airflowとの違いを確認していきます。

### 検証環境

- Docker Desktop
- PostgreSQL 18
- Prefect 3.x（prefecthq/prefect:3-latest）

### 検証リポジトリ

https://github.com/toshiro3/workflow-orchestration-lab/tree/v5-prefect

## Prefectとは

Prefectは、Pythonベースのワークフローオーケストレーションツールです。2018年に創業され、「既存のPythonコードをそのまま使える」ことをコンセプトにしています。

### 特徴

- **DAG定義が不要** - Pythonの制御フローをそのまま使える
- **シンプルなデコレータ** - `@flow`と`@task`を追加するだけ
- **軽量な構成** - 最小1コンテナ（SQLite内蔵）で動作可能

### 3ツールの思想の違い

| ツール | 中心概念 | 特徴 |
|--------|----------|------|
| **Prefect** | Task / Flow | DAG不要、Pythonの制御フローそのまま |
| **Dagster** | Asset | データ資産中心、依存は引数で自動解決 |
| **Airflow** | Task / DAG | 明示的なDAG定義が必要 |

## 環境構築

### アーキテクチャ

Prefectの構成はシンプルです。

**最小構成（2コンテナ）**
- PostgreSQL - メタデータDB
- Prefect Server - APIサーバー + UI

**本番構成（3コンテナ）**
- PostgreSQL
- Prefect Server
- Prefect Worker - Flow実行

Dagster（4コンテナ）やAirflow（5コンテナ）と比較して、コンテナ数が少ないのが特徴です。

また、Airflowの場合はCeleryExecutorを使うとRedisが必要になりますが、Prefectは単一サーバー構成であればRedisは不要です（複数サーバーでの高可用性構成の場合のみ必要）。

### ディレクトリ構成

```
prefect/
├── compose.yaml
├── flows/
│   ├── hello_flow.py
│   ├── csv_pipeline.py
│   ├── csv_pipeline_serve.py
│   └── csv_pipeline_deploy.py
└── data/
    └── sample.csv
```

### compose.yaml（本番構成）

```yaml
services:
  postgres:
    image: postgres:18
    environment:
      POSTGRES_USER: prefect
      POSTGRES_PASSWORD: prefect
      POSTGRES_DB: prefect
    volumes:
      - postgres_data:/var/lib/postgresql  # PostgreSQL 18はマウントポイントに注意
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U prefect"]
      interval: 5s
      timeout: 5s
      retries: 5

  prefect-server:
    image: prefecthq/prefect:3-latest
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      PREFECT_API_DATABASE_CONNECTION_URL: postgresql+asyncpg://prefect:prefect@postgres:5432/prefect
      PREFECT_SERVER_API_HOST: 0.0.0.0
      PREFECT_UI_API_URL: http://localhost:4200/api
    ports:
      - "4200:4200"
    command: prefect server start
    volumes:
      - ./flows:/app/flows
      - ./data:/app/data
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:4200/api/health')"]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 10s

  prefect-worker:
    image: prefecthq/prefect:3-latest
    depends_on:
      prefect-server:
        condition: service_healthy
    environment:
      PREFECT_API_URL: http://prefect-server:4200/api
    command: prefect worker start --pool default-pool --type process
    working_dir: /app/flows
    volumes:
      - ./flows:/app/flows
      - ./data:/app/data

volumes:
  postgres_data:
```

#### ポイント

- **PostgreSQL 18のマウントポイント**: 18以降は`/var/lib/postgresql/data`ではなく`/var/lib/postgresql`にマウントします。18以降ではデータがバージョンごとのサブディレクトリに配置される形式に変更されており、`/data`まで指定するとエラーでコンテナが停止します。この変更により、将来の`pg_upgrade`時にマウントポイントの境界を越えずにデータを移行できるようになっています
- **PREFECT_UI_API_URL**: ブラウザ（クライアント側）がPrefect APIと通信するためのエンドポイントです。`localhost:4200`は「ブラウザを実行しているマシンから見たURL」なので、リモートサーバーに構築する場合は適切なホスト名に変更が必要です
- **healthcheck**: Prefectイメージには`curl`が含まれていないため、Pythonでヘルスチェックを行います
- **service_healthy**: WorkerがServerに接続する前に、Serverの起動完了を待つ設定です

### 起動

```bash
cd prefect
docker compose up -d
docker compose ps
```

```
NAME                       IMAGE                        COMMAND                  SERVICE          CREATED          STATUS                    PORTS
prefect-postgres-1         postgres:18                  "docker-entrypoint.s…"   postgres         17 seconds ago   Up 16 seconds (healthy)   5432/tcp
prefect-prefect-server-1   prefecthq/prefect:3-latest   "/usr/bin/tini -g --…"   prefect-server   17 seconds ago   Up 10 seconds (healthy)   0.0.0.0:4200->4200/tcp
prefect-prefect-worker-1   prefecthq/prefect:3-latest   "/usr/bin/tini -g --…"   prefect-worker   17 seconds ago   Up 5 seconds
```

`http://localhost:4200`にアクセスすると、Prefect UIが表示されます。

## 基本概念 - FlowとTask

Prefectの基本は`@flow`と`@task`デコレータです。

### 最小構成のFlow

```python
# flows/hello_flow.py
from prefect import flow, task

@task
def say_hello(name: str) -> str:
    message = f"Hello, {name}!"
    print(message)
    return message

@task
def say_goodbye(name: str) -> str:
    message = f"Goodbye, {name}!"
    print(message)
    return message

@flow(name="hello-flow")
def hello_flow(name: str = "World"):
    hello_result = say_hello(name)
    goodbye_result = say_goodbye(name)
    return f"{hello_result} / {goodbye_result}"

if __name__ == "__main__":
    hello_flow()
```

### 実行

```bash
docker compose exec prefect-server python /app/flows/hello_flow.py
```

```
14:51:38.349 | INFO    | Flow run 'mutant-sawfish' - Beginning flow run 'mutant-sawfish' for flow 'hello-flow'
Hello, World!
14:51:38.361 | INFO    | Task run 'say_hello-de0' - Finished in state Completed()
Goodbye, World!
14:51:38.363 | INFO    | Task run 'say_goodbye-9d3' - Finished in state Completed()
14:51:38.380 | INFO    | Flow run 'mutant-sawfish' - Finished in state Completed()
```

UIの「Flows」画面で実行履歴が確認できます。

### 3ツールの定義方法比較

**Prefect**
```python
from prefect import flow, task

@task
def extract():
    return data

@task
def transform(data):
    return cleaned

@flow(name="my-pipeline")
def pipeline():
    data = extract()
    result = transform(data)
    return result
```

**Dagster**
```python
from dagster import asset

@asset
def raw_data():
    return data

@asset
def cleaned_data(raw_data):
    return cleaned
```

**Airflow**
```python
from airflow.decorators import dag, task

@task
def extract():
    return data

@task
def transform(data):
    return cleaned

@dag(schedule="@daily")
def pipeline():
    data = extract()
    result = transform(data)
```

Prefectは関数の呼び出し順序がそのまま依存関係になります。Dagsterは引数で依存を表現し、Airflowは`>>`演算子またはTaskFlow APIで定義します。

## CSVパイプライン実装

Dagster・Airflowと同じCSVパイプラインを実装します。

### サンプルデータ

```csv
id,name,email,age,purchase_amount,purchase_date
1,Alice,alice@example.com,28,150.00,2024-01-15
2,Bob,,35,200.50,2024-01-16
3,Charlie,charlie@test.com,42,75.25,2024-01-17
4,Diana,diana@example.com,,300.00,2024-01-18
5,Eve,invalid-email,31,125.75,2024-01-19
6,Frank,frank@example.com,29,,2024-01-20
7,,grace@example.com,38,450.00,2024-01-21
8,Henry,henry@test.com,45,90.00,
```

### パイプラインコード

```python
# flows/csv_pipeline.py
import pandas as pd
from prefect import flow, task
from pathlib import Path

DATA_DIR = Path("/app/data")

@task
def read_csv() -> pd.DataFrame:
    """CSVファイルを読み込む"""
    file_path = DATA_DIR / "sample.csv"
    df = pd.read_csv(file_path)
    print(f"読み込み完了: {len(df)}行")
    print(df.to_string())
    return df

@task
def cleanse_data(df: pd.DataFrame) -> pd.DataFrame:
    """データをクレンジングする"""
    original_count = len(df)
    
    # 欠損値を含む行を削除
    df_cleaned = df.dropna()
    
    # メールアドレスの簡易バリデーション
    df_cleaned = df_cleaned[df_cleaned["email"].str.contains("@", na=False)]
    
    removed_count = original_count - len(df_cleaned)
    print(f"クレンジング完了: {removed_count}行削除, {len(df_cleaned)}行残存")
    print(df_cleaned.to_string())
    return df_cleaned

@task
def aggregate_data(df: pd.DataFrame) -> dict:
    """データを集計する"""
    result = {
        "total_records": len(df),
        "total_purchase_amount": float(df["purchase_amount"].sum()),
        "average_age": float(df["age"].mean()),
        "average_purchase": float(df["purchase_amount"].mean()),
    }
    print(f"集計完了: {result}")
    return result

@flow(name="csv-pipeline")
def csv_pipeline():
    """CSV処理パイプライン"""
    df = read_csv()
    df_cleaned = cleanse_data(df)
    result = aggregate_data(df_cleaned)
    return result

if __name__ == "__main__":
    csv_pipeline()
```

### 実行

```bash
# pandasのインストールが必要
docker compose exec prefect-server pip install pandas
docker compose exec prefect-worker pip install pandas

# 実行
docker compose exec prefect-server python /app/flows/csv_pipeline.py
```

```
読み込み完了: 8行
   id     name              email   age  purchase_amount purchase_date
0   1    Alice  alice@example.com  28.0           150.00    2024-01-15
...
クレンジング完了: 6行削除, 2行残存
   id     name              email   age  purchase_amount purchase_date
0   1    Alice  alice@example.com  28.0           150.00    2024-01-15
2   3  Charlie   charlie@test.com  42.0            75.25    2024-01-17
集計完了: {'total_records': 2, 'total_purchase_amount': 225.25, 'average_age': 35.0, 'average_purchase': 112.625}
```

UIで`csv-pipeline`の実行結果が確認でき、`read_csv` → `cleanse_data` → `aggregate_data`の順序でタスクが実行されたことがタイムラインで可視化されます。

## Deploymentの2つの方式

Prefectでは、Flowを本番運用するために「Deployment」として登録します。登録方法は2つあります。

### 1. .serve() 方式

Pythonプロセスが常駐して実行を待機する方式です。

```python
# flows/csv_pipeline_serve.py
@flow(name="csv-pipeline-scheduled")
def csv_pipeline():
    # ... 省略 ...

if __name__ == "__main__":
    csv_pipeline.serve(
        name="csv-pipeline-deployment",
        cron="*/5 * * * *",
        tags=["csv", "etl"],
    )
```

```bash
docker compose exec -w /app/flows -e PREFECT_API_URL=http://localhost:4200/api prefect-server python csv_pipeline_serve.py
```

```
Your flow 'csv-pipeline-scheduled' is being served and polling for scheduled runs!
```

このコマンドは終了せず、実行待機状態になります。UIの「Deployments」に登録され、スケジュール実行やUIからの手動実行が可能になります。

**注意点**
- `PREFECT_API_URL`環境変数の設定が必要
- 作業ディレクトリ（`-w`オプション）の指定が必要
- プロセスを停止するとDeploymentは「Not Ready」になる

### 2. .deploy() + Worker 方式（本番推奨）

Deploymentを登録だけして、実行はWorkerに任せる方式です。

```python
# flows/csv_pipeline_deploy.py
@flow(name="csv-pipeline-worker")
def csv_pipeline():
    # ... 省略 ...

if __name__ == "__main__":
    csv_pipeline.from_source(
        source="/app/flows",
        entrypoint="csv_pipeline_deploy.py:csv_pipeline",
    ).deploy(
        name="csv-pipeline-worker-deployment",
        work_pool_name="default-pool",
        cron="*/5 * * * *",
        tags=["csv", "etl", "worker"],
    )
```

```bash
docker compose exec -w /app/flows -e PREFECT_API_URL=http://prefect-server:4200/api prefect-server python csv_pipeline_deploy.py
```

```
Successfully created/updated all deployments!

                               Deployments
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┓
┃ Name                                               ┃ Status  ┃ Details ┃
┡━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━┩
│ csv-pipeline-worker/csv-pipeline-worker-deployment │ applied │         │
└────────────────────────────────────────────────────┴─────────┴─────────┘
```

**ポイント**
- コマンドは登録して**終了**する（常駐しない）
- Workerが稼働していれば、Deploymentは「Ready」状態
- スケジュール実行やUIからの手動実行はWorkerが処理

### 2つの方式の比較

| 項目 | `.serve()` | `.deploy()` + Worker |
|------|------------|---------------------|
| Deployment登録 | 登録と同時に待機開始 | 登録して終了 |
| 実行主体 | サーブプロセス自身 | 別プロセスのWorker |
| プロセス停止時 | Not Ready | Ready（Worker稼働中なら） |
| 用途 | 開発・小規模 | 本番運用 |

## Work Pool + Worker構成

`.deploy()`方式では、Work PoolとWorkerの概念が重要です。

### アーキテクチャ

```
Deployment登録
    ↓
Prefect Server（Work Pool: default-pool）
    ↓ ポーリング（デフォルト15秒間隔）
Worker
    ↓
Flow実行
```

WorkerはServerのAPIを定期的にポーリングして、実行すべきFlow Runを取得します。AirflowのCeleryExecutorのようなメッセージブローカーは不要です。

### UIからの実行

1. 「Deployments」で対象のDeploymentをクリック
2. 右上の「Run」→「Quick run」
3. Workerが実行を取得して処理

Workerのログ:
```
15:32:04.328 | INFO    | prefect.flow_runs.worker - Worker 'ProcessWorker ...' submitting flow run '...'
15:32:04.428 | INFO    | prefect.flow_runs.runner - Opening process...
読み込み完了: 8行
クレンジング完了: 6行削除, 2行残存
集計完了: {'total_records': 2, 'total_purchase_amount': 225.25, 'average_age': 35.0, 'average_purchase': 112.625}
15:32:05.371 | INFO    | Flow run 'miniature-flounder' - Finished in state Completed()
```

### Work Pools画面

UIの「Work Pools」で以下が確認できます:
- Pool名（default-pool）
- タイプ（Process）
- Workerの稼働状況（Last Polled）
- 同時実行数制限（Concurrency Limit）

## デプロイ方法の違い

3ツールで新しいワークフローを追加する際の違いを整理します。

| ツール | 新しいワークフロー追加時 |
|--------|-------------------------|
| **Prefect** | `prefect deploy`コマンドの実行が必要 |
| **Dagster** | ファイル配置で自動認識 |
| **Airflow** | ファイル配置で自動認識 |

Prefectは、新しいFlowを追加した際に`prefect deploy`コマンドを実行してDeploymentを登録する必要があります。CI/CDで自動化する場合は、このステップをワークフローに組み込みます。

ただし、一度Deploymentを登録すれば、**関数の中身を書き換えるだけなら再デプロイは不要**です。Workerが実行時にソースコード（`/app/flows`など）から最新を読み込むためです。関数のシグネチャやスケジュールが変わる場合のみ再デプロイが必要になります。

## UIの比較

| 項目 | Prefect | Dagster | Airflow |
|------|---------|---------|---------|
| **デフォルトポート** | 4200 | 3000 | 8080 |
| **実行履歴** | Runs画面（タイムライン） | Runs画面（ステップ） | Grid/Graph/Gantt |
| **手動実行** | Deployment → Quick run | Materialize | Trigger DAG |
| **依存関係可視化** | タイムライン / Radar View | Asset Lineage | Graph View |

Prefect UIはシンプルでモダンな印象です。タイムライン表示でタスクの実行順序が直感的に把握できます。また、Radar Viewではタスク間の依存グラフを視覚的に確認できます。

## まとめ

### アーキテクチャ比較

| 項目 | Prefect | Dagster | Airflow |
|------|---------|---------|---------|
| **最小構成** | 2コンテナ | 4コンテナ | 5コンテナ |
| **本番構成** | 3コンテナ | 4コンテナ | 5コンテナ |
| **メッセージブローカー** | 不要 | 不要 | Executor次第 |

### 定義方法比較

| ツール | デコレータ | 依存関係 |
|--------|-----------|----------|
| **Prefect** | `@flow` / `@task` | 関数呼び出し順序 |
| **Dagster** | `@asset` / `@op` | 関数引数 |
| **Airflow** | `@dag` / `@task` | `>>` or TaskFlow |

### Prefectの特徴

**構成面**
- 最小1コンテナ（SQLite）で動作可能
- 本番でも3コンテナとコンパクト
- Redisは複数Server構成でのみ必要

**開発面**
- DAG定義不要、Pythonの制御フローそのまま
- `@flow`と`@task`デコレータを追加するだけ
- ローカルで`python flow.py`ですぐ動く

**運用面**
- `.serve()`（開発向け）と`.deploy()`（本番向け）の2つの方式
- 新しいFlowの追加時は`prefect deploy`コマンドの実行が必要
- Work Pool + Workerでスケーラブルな構成が可能

### 検証を終えて

Prefectは「既存のPythonコードをそのまま使える」というコンセプト通り、デコレータを追加するだけでワークフロー化できる手軽さが魅力です。コンテナ構成もシンプルで、学習コストは比較的低いと感じました。

3ツールそれぞれに特徴があり、用途やチームの状況に応じて選択することになります。

## 参考リンク

- [Prefect公式ドキュメント](https://docs.prefect.io/)
- [Prefect GitHub](https://github.com/PrefectHQ/prefect)
- [検証リポジトリ](https://github.com/toshiro3/workflow-orchestration-lab)
