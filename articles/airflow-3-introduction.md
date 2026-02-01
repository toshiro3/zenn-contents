---
title: "Apache Airflow 3.xをDocker Composeで動かしてみた - 環境構築からCSVパイプライン実装まで"
emoji: "🌀"
type: "tech"
topics: ["airflow", "docker", "python", "データエンジニアリング", "ワークフロー"]
published: true
---

## はじめに

Apache Airflow 3.0が2025年4月にGA（正式リリース）されました。2020年の2.0リリースから約5年ぶりのメジャーアップデートであり、**Airflow史上最大のリリース**と言われています。

以前の記事でDagsterを検証したので、今回はAirflow 3.xをローカルで動かして比較してみることにしました。実際にDocker Compose環境を構築し、CSVパイプラインを実装するところまで検証した記録です。

https://zenn.dev/toshiro3/articles/dagster-introduction

### 検証環境

| コンポーネント | バージョン |
|--------------|----------|
| Apache Airflow | 3.1.5 |
| PostgreSQL | 18 |
| Docker Compose | v2.14.0+ |
| Executor | LocalExecutor |

:::message
本記事のコードは以下のリポジトリで公開しています。
https://github.com/toshiro3/workflow-orchestration-lab/tree/v4-airflow
:::

## Airflow 3.0の特徴

まず、3.0の特徴を調べてみました。

### Task Execution API の導入（AIP-72）

3.0の大きな特徴です。Schedulerとタスク実行が分離されており、タスクはTask Execution APIを通じて実行されます。

```
┌─────────────────────────────────────────────────────────────────┐
│                        Docker Compose                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    REST API    ┌──────────────────┐          │
│  │  Scheduler   │ ─────────────▶ │   API Server     │ :8080    │
│  └──────────────┘    (JWT認証)    │ (Execution API)  │          │
│         │                        └──────────────────┘          │
│         │                                 │                     │
│         ▼                                 ▼                     │
│  ┌──────────────┐                ┌──────────────────┐          │
│  │ DAG Processor│                │   PostgreSQL     │          │
│  └──────────────┘                │  (メタデータDB)   │          │
│         │                        └──────────────────┘          │
│         ▼                                                       │
│  ┌──────────────┐                                              │
│  │  Triggerer   │                                              │
│  └──────────────┘                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

後述しますが、このアーキテクチャのせいで環境構築時にいくつかハマりました。

### 新しい名前空間 `airflow.sdk`

DAG作成者向けに安定したインターフェースが提供されるようになりました。

```python
# Airflow 3.x の推奨インポート
from airflow.sdk import dag, task

# Airflow 2.x のインポート（後方互換あり）
from airflow.decorators import dag, task
```

新しいSDKは、重いAirflow本体をインポートせずにDAGを定義できるため、パースの高速化とメモリ節約に寄与します。

### DAG Processorの独立

DAGファイルのパース処理が独立したコンポーネントになっています。`airflow-dag-processor`という別コンテナで動きます。

### React UIへの完全刷新

UIがReactベースで完全に書き直されていました。左サイドバーに「Assets」メニューが追加されるなど、見た目も操作感もかなり変わっています。

![Airflow 3.x ダッシュボード](/images/airflow-3-introduction/airflow-dashboard.png)
*Airflow 3.x のダッシュボード画面*

### Asset-Based Scheduling

`@asset`デコレータでデータ駆動型のパイプラインを定義できるようになっています。これはDagsterの影響を感じる機能ですね。

## 環境構築

### ディレクトリ構成

以下の構成で進めました。

```
airflow/
├── compose.yaml      # Docker Compose設定
├── .env              # 環境変数
├── dags/             # DAGファイル
├── logs/             # ログ出力
├── plugins/          # カスタムプラグイン
├── config/           # 設定ファイル
└── data/             # 検証用データ
```

:::message
ファイル名について、Docker公式ドキュメントでは`compose.yaml`が推奨されています（`docker-compose.yaml`は後方互換）。今回は新しい形式を採用しました。
:::

### compose.yamlの作成

公式のdocker-compose.yamlはCeleryExecutor構成で複雑だったので、**LocalExecutor**でシンプルに構成しました。

:::details compose.yaml（クリックで展開）
```yaml
x-airflow-common:
  &airflow-common
  image: apache/airflow:3.1.5
  environment:
    &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: LocalExecutor
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
    AIRFLOW__CORE__FERNET_KEY: ${AIRFLOW__CORE__FERNET_KEY}  # .envで定義
    AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
    AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    AIRFLOW__SCHEDULER__ENABLE_HEALTH_CHECK: 'true'
    AIRFLOW__API__AUTH_BACKENDS: 'airflow.providers.fab.auth_manager.api.auth.backend.basic_auth,airflow.api.auth.backend.session'
    # Airflow 3.0 固有の設定
    AIRFLOW__CORE__AUTH_MANAGER: airflow.providers.fab.auth_manager.fab_auth_manager.FabAuthManager
    AIRFLOW__CORE__EXECUTION_API_SERVER_URL: http://airflow-api-server:8080/execution/
    AIRFLOW__API_AUTH__JWT_SECRET: 'your-super-secret-jwt-key-change-in-production'
  volumes:
    - ./dags:/opt/airflow/dags
    - ./logs:/opt/airflow/logs
    - ./config:/opt/airflow/config
    - ./plugins:/opt/airflow/plugins
    - ./data:/opt/airflow/data
  user: "${AIRFLOW_UID:-50000}:0"
  depends_on:
    &airflow-common-depends-on
    postgres:
      condition: service_healthy

services:
  postgres:
    image: postgres:18
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - postgres-db-volume:/var/lib/postgresql
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "airflow"]
      interval: 10s
      retries: 5
      start_period: 5s
    restart: always

  airflow-api-server:
    <<: *airflow-common
    command: api-server
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8080/api/v1/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-scheduler:
    <<: *airflow-common
    command: scheduler
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8974/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-dag-processor:
    <<: *airflow-common
    command: dag-processor
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8794/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-triggerer:
    <<: *airflow-common
    command: triggerer
    healthcheck:
      test: ["CMD-SHELL", 'airflow jobs check --job-type TriggererJob --hostname "$${HOSTNAME}"']
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-init:
    <<: *airflow-common
    entrypoint: /bin/bash
    command:
      - -c
      - |
        mkdir -p /opt/airflow/{logs,dags,plugins}
        chown -R "${AIRFLOW_UID}:0" /opt/airflow/{logs,dags,plugins}
        exec /entrypoint airflow db migrate && airflow users create \
          --username airflow \
          --firstname Admin \
          --lastname User \
          --role Admin \
          --email admin@example.com \
          --password airflow
    environment:
      <<: *airflow-common-env
      _AIRFLOW_DB_MIGRATE: 'true'
      _AIRFLOW_WWW_USER_CREATE: 'true'
      _AIRFLOW_WWW_USER_USERNAME: airflow
      _AIRFLOW_WWW_USER_PASSWORD: airflow
    user: "0:0"
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  postgres-db-volume:
```
:::

### 環境構築でハマったポイント

ここからが本題です。環境構築時にいくつかのトラブルに遭遇しました。

#### ハマり1: PostgreSQL 18の起動失敗

PostgreSQL 18を使おうとしたら、以下のエラーで起動しませんでした。

```
Error: in 18+, these Docker images are configured to store database data...
```

**原因**: PostgreSQL 18からデータディレクトリの構造が変更されていました。

**解決策**: ボリュームのマウント先を変更

```yaml
# 変更前
volumes:
  - postgres-db-volume:/var/lib/postgresql/data

# 変更後
volumes:
  - postgres-db-volume:/var/lib/postgresql  # /data を削除
```

:::message
別の方法として、`PGDATA`環境変数を指定する手法もあります。既存の運用ルールで `/var/lib/postgresql/data` を維持したい場合はこちらが便利です。
```yaml
environment:
  PGDATA: /var/lib/postgresql/data/pgdata
```
:::

#### ハマり2: タスク実行時のConnection refused

UIは起動したものの、DAGを実行すると以下のエラーが発生しました。

```
httpcore.ConnectError: [Errno 111] Connection refused
```

**原因**: Task Execution API URLが設定されていませんでした。3.0ではSchedulerがAPI Serverと通信してタスクを実行するため、この設定が必須です。

**解決策**: 環境変数を追加

```yaml
AIRFLOW__CORE__EXECUTION_API_SERVER_URL: http://airflow-api-server:8080/execution/
```

#### ハマり3: JWT認証エラー

接続はできるようになったものの、今度は認証エラーが発生しました。

```
ServerResponseError: Invalid auth token: Signature verification failed
```

**原因**: Task Execution APIの内部通信でJWT認証が使用されるようになっており、シークレットの設定が必要でした。

**解決策**: JWT Secretを追加

```yaml
AIRFLOW__API_AUTH__JWT_SECRET: 'your-super-secret-jwt-key-change-in-production'
```

#### ハマり4: ログインできない

UIは表示されるのに、`airflow` / `airflow`でログインできませんでした。画面には「Simple auth manager enabled」と表示されています。

**原因**: 3.0からSimple Auth Managerがデフォルトになっており、FAB Auth Managerを使うには明示的な設定が必要でした。

**解決策**: Auth Managerを指定

```yaml
AIRFLOW__CORE__AUTH_MANAGER: airflow.providers.fab.auth_manager.fab_auth_manager.FabAuthManager
```

### 最終的に必要だった3.0固有の設定

試行錯誤の結果、以下の設定が必要だと分かりました。

| 設定項目 | 値 | 理由 |
|---------|-----|------|
| `AIRFLOW__CORE__AUTH_MANAGER` | FabAuthManager | FAB Auth Managerでユーザー認証 |
| `AIRFLOW__CORE__EXECUTION_API_SERVER_URL` | `http://airflow-api-server:8080/execution/` | Scheduler → API Server通信 |
| `AIRFLOW__API_AUTH__JWT_SECRET` | 任意の文字列 | Task Execution APIのJWT認証 |

「Airflow 3.0 docker-compose EXECUTION_API_SERVER_URL」などで検索し、GitHubのIssueや他の方の記事を参考に解決しました。

### 起動手順

```bash
# ディレクトリ作成
mkdir -p airflow/dags airflow/logs airflow/plugins airflow/config airflow/data
cd airflow

# .envファイル作成（Linux）
echo "AIRFLOW_UID=$(id -u)" > .env

# FERNET_KEYを生成して.envに追加
echo "AIRFLOW__CORE__FERNET_KEY=$(openssl rand -base64 32)" >> .env

# 初期化
docker compose up airflow-init

# 全サービス起動
docker compose up -d

# 状態確認
docker compose ps
```

:::message
`FERNET_KEY`はConnectionsなどの機密情報を暗号化するために使用されます。空のままだとConnectionsが正しく保存できない場合があるため、必ず設定してください。
:::

http://localhost:8080 にアクセスし、`airflow` / `airflow` でログインできれば成功です。

### コンテナ構成（5コンテナ）

最終的なコンテナ構成は以下のようになりました。

| コンテナ | 役割 |
|---------|------|
| postgres | メタデータDB |
| airflow-api-server | Web UI + REST API（ポート8080） |
| airflow-scheduler | タスクスケジューリング |
| airflow-dag-processor | DAGファイルのパース |
| airflow-triggerer | Deferrable tasks用 |

CeleryExecutor構成だとredis、airflow-worker、flowerも必要ですが、LocalExecutorなら上記5つで動きます。

## Hello World DAGで動作確認

まずはシンプルなDAGで動作確認しました。

```python:dags/hello_world.py
from airflow.sdk import dag, task
from datetime import datetime


@dag(
    schedule=None,
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=["example"],
)
def hello_world():
    """最初のDAG - Hello World"""

    @task
    def say_hello():
        print("Hello, Airflow 3!")
        return "Hello"

    @task
    def say_goodbye(greeting: str):
        print(f"{greeting} and Goodbye!")
        return "Done"

    result = say_hello()
    say_goodbye(result)


hello_world()
```

ファイルを保存すると、DAG Processorが自動的に検出してくれます。検出状況は以下のコマンドで確認できました。

```bash
docker compose logs airflow-dag-processor -f
```

`Found X files for bundle dags-folder` と表示されればDAGが認識されています。

UIでDAGを有効化（トグルをON）し、再生ボタンをクリックして実行。無事成功しました。

![hello_world DAG 実行成功](/images/airflow-3-introduction/hello-world-success.jpg)
*hello_world DAGの実行結果*

## CSVパイプラインの実装

次に、Dagsterの記事で実装したのと同等の「CSV読み込み → クレンジング → 集計」パイプラインを作成しました。

### サンプルデータ

```csv:data/sales_raw.csv
order_id,product_name,quantity,unit_price,order_date
1,Apple,10,150,2024-01-15
2,Banana,5,100,2024-01-15
3,Orange,NULL,200,2024-01-16
4,Apple,-3,150,2024-01-16
5,Banana,8,100,2024-01-17
6,Grape,12,300,2024-01-17
7,Orange,6,,2024-01-18
8,Apple,15,150,2024-01-18
```

クレンジング対象として、NULL値、負の数、空値を含めています。

### DAGファイル

```python:dags/csv_pipeline.py
"""
CSV Pipeline - Airflow 3.x版
処理フロー:
1. extract: CSVファイルを読み込み
2. cleanse: データクレンジング（NULL除去、負値除去）
3. aggregate: 商品別集計
"""

from airflow.sdk import dag, task
from datetime import datetime
import csv
import json
from pathlib import Path


DATA_DIR = Path("/opt/airflow/data")


@dag(
    schedule=None,
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=["csv", "pipeline", "example"],
    description="CSV読み込み → クレンジング → 集計パイプライン",
)
def csv_pipeline():
    """Dagster版と同等のCSV処理パイプライン"""

    @task
    def extract() -> list[dict]:
        """CSVファイルを読み込む"""
        input_path = DATA_DIR / "sales_raw.csv"
        
        with open(input_path, "r", encoding="utf-8") as f:
            reader = csv.DictReader(f)
            records = list(reader)
        
        print(f"読み込み件数: {len(records)}")
        print(f"入力ファイル: {input_path}")
        
        return records

    @task
    def cleanse(raw_data: list[dict]) -> list[dict]:
        """データクレンジング
        - NULL/空値を含む行を除去
        - 負の数量を除去
        """
        cleansed = []
        removed_count = 0
        
        for row in raw_data:
            # NULL または空値チェック
            if row["quantity"] in ("NULL", "", None):
                removed_count += 1
                continue
            if row["unit_price"] in ("NULL", "", None):
                removed_count += 1
                continue
            
            # 数値変換と負値チェック
            quantity = int(row["quantity"])
            if quantity < 0:
                removed_count += 1
                continue
            
            cleansed.append({
                "order_id": int(row["order_id"]),
                "product_name": row["product_name"],
                "quantity": quantity,
                "unit_price": int(row["unit_price"]),
                "order_date": row["order_date"],
            })
        
        print(f"クレンジング前: {len(raw_data)}件")
        print(f"除去件数: {removed_count}件")
        print(f"クレンジング後: {len(cleansed)}件")
        
        return cleansed

    @task
    def aggregate(cleansed_data: list[dict]) -> dict:
        """商品別に集計"""
        summary = {}
        
        for row in cleansed_data:
            product = row["product_name"]
            amount = row["quantity"] * row["unit_price"]
            
            if product not in summary:
                summary[product] = {
                    "total_quantity": 0,
                    "total_amount": 0,
                    "order_count": 0,
                }
            
            summary[product]["total_quantity"] += row["quantity"]
            summary[product]["total_amount"] += amount
            summary[product]["order_count"] += 1
        
        # 結果をファイルに出力
        output_path = DATA_DIR / "sales_summary.json"
        with open(output_path, "w", encoding="utf-8") as f:
            json.dump(summary, f, indent=2, ensure_ascii=False)
        
        print(f"集計結果: {json.dumps(summary, indent=2, ensure_ascii=False)}")
        print(f"出力ファイル: {output_path}")
        
        return summary

    # タスクの依存関係を定義（TaskFlow API）
    raw = extract()
    cleansed = cleanse(raw)
    aggregate(cleansed)


csv_pipeline()
```

### 実行結果

DAGを実行すると、以下のような集計結果が生成されました。

```json:data/sales_summary.json
{
  "Apple": {
    "total_quantity": 25,
    "total_amount": 3750,
    "order_count": 2
  },
  "Banana": {
    "total_quantity": 13,
    "total_amount": 1300,
    "order_count": 2
  },
  "Grape": {
    "total_quantity": 12,
    "total_amount": 3600,
    "order_count": 1
  }
}
```

元データ8件からNULL値、空値、負値を含む3件が除去され、5件が集計されています。想定通りの結果になりました。

![csv_pipeline DAG 実行成功](/images/airflow-3-introduction/csv-pipeline-success.jpg)
*csv_pipeline DAGの実行結果*

## Dagsterと比較してみた

同じパイプラインを両方で実装したので、比較してみます。

### コードの比較

**Dagster（Asset）**
```python
@asset
def raw_sales_data():
    return pd.read_csv("sales_raw.csv")

@asset(deps=[raw_sales_data])
def cleansed_sales_data(raw_sales_data):
    return raw_sales_data.dropna()

@asset(deps=[cleansed_sales_data])
def sales_summary(cleansed_sales_data):
    return cleansed_sales_data.groupby("product").sum()
```

**Airflow 3.x（TaskFlow API）**
```python
@dag(schedule=None, start_date=datetime(2024, 1, 1))
def csv_pipeline():
    @task
    def extract() -> list[dict]:
        return list(csv.DictReader(open("sales_raw.csv")))

    @task
    def cleanse(raw_data: list[dict]) -> list[dict]:
        return [r for r in raw_data if valid(r)]

    @task
    def aggregate(cleansed_data: list[dict]) -> dict:
        return summary

    raw = extract()
    cleansed = cleanse(raw)
    aggregate(cleansed)

csv_pipeline()
```

### 主な違い

| 項目 | Dagster | Airflow 3.x |
|------|---------|-------------|
| 中心概念 | Asset（データ中心） | Task（処理中心） |
| デコレータ | `@asset` | `@task` + `@dag` |
| 依存関係の定義 | `deps`パラメータ | 関数呼び出しチェーン |
| データ受け渡し | I/O Manager経由 | XCom（自動） |
| DAG定義 | 暗黙的（依存関係から推論） | 明示的（関数内で定義） |

Dagsterは「何を作るか（Asset）」、Airflowは「何をするか（Task）」という設計思想の違いを感じました。

### アーキテクチャの比較

**Dagster**
- gRPCでUser Codeを分離

**Airflow 3.x**
- Task Execution APIでタスク実行を分離
- JWT認証による内部通信
- DAG Processorが独立コンポーネント

どちらもコード実行を分離する方向に進化していますが、アプローチが異なります。

### 環境構築の比較

| 項目 | Dagster | Airflow 3.x |
|------|---------|-------------|
| コンテナ数（LocalExecutor相当） | 4 | 5 |
| 初期設定の複雑さ | 低 | 高（JWT、Auth Manager等） |
| 公式Docker Compose | シンプル | 設定項目多め |

正直なところ、Dagsterの方が環境構築は楽でした。Airflow 3.xは設定項目が多く、動かすまでに少し苦労しました。

### 使い分けの所感

**Dagsterが向いていそう**
- データパイプラインが中心
- データリネージュの可視化が重要
- 新規プロジェクトで導入

**Airflowが向いていそう**
- 汎用的なワークフロー管理
- 既存のAirflow資産がある
- 豊富なOperator/Providerを活用したい
- 大規模な本番運用実績が必要

## まとめ

Airflow 3.xを実際に動かしてみました。

### 環境構築時の注意点

| 設定項目 | 値 | 理由 |
|---------|-----|------|
| `AIRFLOW__CORE__AUTH_MANAGER` | FabAuthManager | FAB Auth Managerでユーザー認証 |
| `AIRFLOW__CORE__EXECUTION_API_SERVER_URL` | `http://airflow-api-server:8080/execution/` | Scheduler → API Server通信 |
| `AIRFLOW__API_AUTH__JWT_SECRET` | 任意の文字列 | Task Execution APIのJWT認証 |
| PostgreSQL volumes | `/var/lib/postgresql` | PostgreSQL 18の新ディレクトリ構造 |

新規プロジェクトでは3.x系から始めるのが良さそうです。

### 所感

- Task Execution APIの導入で、将来的にリモート実行やコンテナ実行がやりやすくなりそう
- React UIは見やすくなった
- 環境構築は設定項目が多く、少し複雑な印象
- Dagsterと比較すると、汎用性のAirflow、データ特化のDagsterという棲み分けが見える

## 参考リンク

- [Apache Airflow 3.0 Release Notes](https://airflow.apache.org/docs/apache-airflow/3.0.0/release_notes.html)
- [Running Airflow in Docker](https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html)
- [Apache Airflow® 3 is Generally Available!](https://airflow.apache.org/blog/airflow-three-point-oh-is-here/)
