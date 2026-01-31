---
title: "dagster-dbtでDagsterとdbtを連携させてみた"
emoji: "🔗"
type: "tech"
topics: ["dagster", "dbt", "postgresql", "docker", "dataengineering"]
published: false
---

## はじめに

データ変換ツールとして広く使われている**dbt**と、オーケストレーションツールの**Dagster**を連携させる方法を検証してみました。

`dagster-dbt`パッケージを使うことで、dbtのモデルをDagster Assetsとして扱い、統一的なオーケストレーションが可能になります。実際にDocker Compose環境を構築して、どこまでできるのか試してみます。

### 検証すること

- Docker Composeを使った開発環境の構築
- dbtプロジェクトの作成とモデル定義
- dagster-dbtによるdbtモデルのAsset化
- dbt testのDagsterからの実行
- 部分実行（特定モデルのみの実行）

### 前提知識

- Dagsterの基本的な概念（Asset、Resource）
- dbtの基本的な使い方（models、seeds、profiles）
- Docker Composeの基本操作

### 検証環境

| ツール | バージョン |
|--------|-----------|
| Python | 3.13.11 |
| Dagster | 1.12.13 |
| dagster-dbt | 0.28.13 |
| dbt-core | 1.11.2 |
| dbt-postgres | 1.10.0 |
| PostgreSQL | 17 |

検証用のソースコードはGitHubで公開しています。

https://github.com/toshiro3/dagster-dbt-demo

---

## プロジェクト構成

最終的なディレクトリ構成は以下のようになりました。

```
dagster-dbt-demo/
├── docker/
│   └── dagster/
│       └── Dockerfile
├── dagster_project/
│   ├── __init__.py
│   ├── assets.py
│   ├── definitions.py
│   └── project.py
├── dbt_project/
│   ├── models/
│   │   ├── staging/
│   │   │   ├── schema.yml
│   │   │   ├── stg_orders.sql
│   │   │   └── stg_users.sql
│   │   └── marts/
│   │       └── user_order_summary.sql
│   ├── seeds/
│   │   ├── raw_orders.csv
│   │   └── raw_users.csv
│   ├── dbt_project.yml
│   └── profiles.yml
├── docker-compose.yml
├── pyproject.toml
├── requirements.txt
├── .env
├── .env.example
└── .gitignore
```

Docker Compose関連のファイルは`docker/`ディレクトリにまとめ、Dagsterプロジェクトとdbtプロジェクトを分離した構成にしました。

---

## 環境構築

### 1. Docker Compose設定

まず、`docker-compose.yml`を作成します。PostgreSQLとDagsterの2つのサービスを定義しました。

```yaml:docker-compose.yml
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  dagster:
    build:
      context: .
      dockerfile: docker/dagster/Dockerfile
    ports:
      - "3000:3000"
    volumes:
      - .:/app
    environment:
      POSTGRES_HOST: ${POSTGRES_HOST}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    depends_on:
      - postgres

volumes:
  postgres_data:
```

PostgreSQLはdbtのターゲットDBとして使用します。環境変数は`.env`ファイルから読み込む形式にして、認証情報をハードコードしないようにしています。

### 2. Dockerfile

Dagsterコンテナ用の`Dockerfile`を作成します。

```dockerfile:docker/dagster/Dockerfile
FROM python:3.13-slim

WORKDIR /app

RUN apt-get update && apt-get install -y \
    git \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install -r requirements.txt

CMD ["dagster", "dev", "-h", "0.0.0.0", "-p", "3000"]
```

dbtはGitリポジトリからパッケージをインストールする機能があるため、gitをインストールしています。

### 3. requirements.txt

必要なパッケージを定義します。

```txt:requirements.txt
dagster
dagster-webserver
dagster-dbt
dbt-postgres
```

`dagster-dbt`がDagsterとdbtを連携させるためのパッケージです。`dbt-postgres`はdbtのPostgreSQLアダプターです。

### 4. 環境変数

`.env.example`をテンプレートとして作成し、実際の`.env`ファイルはGit管理外にします。

```env:.env.example
POSTGRES_USER=dbt_user
POSTGRES_PASSWORD=dbt_password
POSTGRES_DB=dbt_db
POSTGRES_HOST=postgres
```

```env:.env
POSTGRES_USER=dbt_user
POSTGRES_PASSWORD=dbt_password
POSTGRES_DB=dbt_db
POSTGRES_HOST=postgres
```

### 5. .gitignore

```gitignore:.gitignore
# Environment
.env

# Python
__pycache__/
*.pyc
.venv/

# dbt
target/
dbt_packages/
logs/

# Dagster
.dagster/
storage/

# IDE
.idea/
.vscode/
```

### 6. ビルドと起動確認

設定ファイルの作成が完了したら、ビルドを実行します。

```bash
docker compose build
```

PostgreSQLが起動できることを確認します。

```bash
docker compose up -d postgres
docker compose ps
```

ここまでは特に問題なく進みました。

---

## dbtプロジェクトの作成

### 1. dbt init

コンテナ内でdbtプロジェクトを初期化します。

```bash
docker compose run --rm dagster dbt init dbt_project
```

対話形式で以下の情報を入力しました。

| 項目 | 入力値 |
|------|--------|
| database | postgres（番号で選択） |
| host | postgres |
| port | 5432 |
| user | dbt_user |
| pass | dbt_password |
| dbname | dbt_db |
| schema | public |
| threads | 1 |

### 2. profiles.yml

`dbt init`で作成された`profiles.yml`はコンテナ内の`/root/.dbt/`に配置されますが、コンテナを再起動すると消えてしまいます。そこで、プロジェクト内に配置し、環境変数を使う形式に変更しました。

```yaml:dbt_project/profiles.yml
dbt_project:
  target: dev
  outputs:
    dev:
      type: postgres
      host: "{{ env_var('POSTGRES_HOST') }}"
      port: 5432
      user: "{{ env_var('POSTGRES_USER') }}"
      password: "{{ env_var('POSTGRES_PASSWORD') }}"
      dbname: "{{ env_var('POSTGRES_DB') }}"
      schema: public
      threads: 1
```

環境変数を使うことで、認証情報をハードコードせずに済みます。

### 3. 接続確認

```bash
docker compose run --rm dagster bash -c "cd dbt_project && dbt debug --profiles-dir ."
```

`All checks passed!`と表示され、接続確認OKです。

### 4. サンプルデータ（seeds）の作成

検証用のサンプルデータを作成します。

```csv:dbt_project/seeds/raw_users.csv
id,name,email,created_at
1,Alice,alice@example.com,2024-01-01
2,Bob,bob@example.com,2024-01-02
3,Charlie,charlie@example.com,2024-01-03
```

```csv:dbt_project/seeds/raw_orders.csv
id,user_id,amount,ordered_at
1,1,1000,2024-01-10
2,1,2000,2024-01-15
3,2,1500,2024-01-12
4,3,3000,2024-01-20
```

### 5. stagingモデルの作成

seedsデータを整形するstagingモデルを作成します。

```sql:dbt_project/models/staging/stg_users.sql
select
    id as user_id,
    name,
    email,
    created_at::date as created_at
from {{ ref('raw_users') }}
```

```sql:dbt_project/models/staging/stg_orders.sql
select
    id as order_id,
    user_id,
    amount,
    ordered_at::date as ordered_at
from {{ ref('raw_orders') }}
```

### 6. martsモデルの作成

stagingモデルを集計するmartsモデルを作成します。

```sql:dbt_project/models/marts/user_order_summary.sql
select
    u.user_id,
    u.name,
    count(o.order_id) as total_orders,
    sum(o.amount) as total_amount
from {{ ref('stg_users') }} u
left join {{ ref('stg_orders') }} o on u.user_id = o.user_id
group by u.user_id, u.name
```

### 7. dbt_project.ymlの修正

dbt initで作成されたデフォルト設定を、今回の構成に合わせて修正します。

```yaml:dbt_project/dbt_project.yml
name: 'dbt_project'
version: '1.0.0'

profile: 'dbt_project'

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

clean-targets:
  - "target"
  - "dbt_packages"

models:
  dbt_project:
    staging:
      +materialized: view
    marts:
      +materialized: table
```

stagingモデルはview、martsモデルはtableとしてマテリアライズする設定にしました。

### 8. 既存のexampleモデルを削除

dbt initで作成されるexampleモデルは不要なので削除します。

```bash
rm -rf dbt_project/models/example
```

### 9. dbt実行確認

dbt単体での動作を確認します。

```bash
docker compose run --rm dagster bash -c "cd dbt_project && dbt seed --profiles-dir . && dbt run --profiles-dir ."
```

```
16:14:44  Done. PASS=2 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=2
...
16:14:47  Done. PASS=3 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=3
```

seedsとmodelsが正常に実行されました。dbt単体では問題なく動作することを確認できました。

---

## Dagsterとdbtの連携

ここからが本題です。`dagster-dbt`パッケージを使って、dbtモデルをDagster Assetsとして扱えるようにします。

### 1. Dagsterプロジェクトの作成

以下の4ファイルを作成しました。

#### dagster_project/\_\_init\_\_.py

空ファイルでOKです。

```python:dagster_project/__init__.py
```

#### dagster_project/project.py

dbtプロジェクトの場所を定義します。

```python:dagster_project/project.py
from pathlib import Path

from dagster_dbt import DbtProject

dbt_project = DbtProject(
    project_dir=Path(__file__).parent.parent / "dbt_project",
    profiles_dir=Path(__file__).parent.parent / "dbt_project",
)
dbt_project.prepare_if_dev()
```

`DbtProject`クラスでdbtプロジェクトのディレクトリとprofiles.ymlの場所を指定します。

`prepare_if_dev()`は開発モード時に`dbt compile`を実行してmanifest.jsonを自動生成してくれるメソッドです。通常、dbtのモデルを変更するたびに手動で`dbt compile`を実行してmanifest.jsonを更新する必要がありますが、このメソッドがあることで「dbt側でモデルを追加・変更した後、Dagster UIの『Reload definitions』ボタンを押すだけで、自動的にmanifest.jsonが再生成され、新しいAssetがUIに反映される」という快適な開発体験が得られます。

#### dagster_project/assets.py

dbtモデルをDagster Assetsとして定義します。

```python:dagster_project/assets.py
from dagster_dbt import DbtCliResource, dbt_assets

from .project import dbt_project


@dbt_assets(manifest=dbt_project.manifest_path)
def my_dbt_assets(context, dbt: DbtCliResource):
    yield from dbt.cli(["build"], context=context).stream()
```

`@dbt_assets`デコレータを使うことで、dbtのmanifest.jsonからモデル情報を読み取り、自動的にDagster Assetsを生成してくれます。

`dbt build`コマンドを使用しているため、モデルの実行（run）とテスト（test）が両方実行されます。実務では、`dbt.cli(["build", "--select", "tag:daily"], context=context)`のようにタグやパスでフィルタリングして、特定のモデルだけを実行対象にすることも可能です。

また、本番環境で実行時間を短縮したい場合は、`dbt build --select state:modified+`（変更があったモデルとその下流のみ）を実行するように構成することもできます。

#### dagster_project/definitions.py

Dagsterの定義をまとめます。

```python:dagster_project/definitions.py
from dagster import Definitions, define_asset_job, in_process_executor
from dagster_dbt import DbtCliResource

from .assets import my_dbt_assets
from .project import dbt_project


dbt_job = define_asset_job(
    name="dbt_job",
    selection="*",
    executor_def=in_process_executor,
)

defs = Definitions(
    assets=[my_dbt_assets],
    jobs=[dbt_job],
    resources={
        "dbt": DbtCliResource(
            project_dir=dbt_project.project_dir,
            profiles_dir=dbt_project.profiles_dir,
        ),
    },
)
```

`DbtCliResource`はdbt CLIを実行するためのResourceです。プロジェクトディレクトリとprofilesディレクトリを指定します。

Docker環境ではマルチプロセス実行でエラーが発生したため、`in_process_executor`を指定したJobを定義しました。UIの「Materialize all」ではなく、このJobから実行する形になります。

:::message
`in_process_executor`はローカル開発には適していますが、本番環境ではスケーラビリティの観点から`docker_executor`や`k8s_job_executor`の使用を検討してください。
:::

### 2. pyproject.toml

Dagsterがプロジェクトを認識するための設定ファイルを作成します。

```toml:pyproject.toml
[tool.dagster]
module_name = "dagster_project.definitions"
```

### 3. Dagster UI起動

```bash
docker compose up dagster
```

http://localhost:3000 にアクセスします。

---

## 動作確認

### Asset Catalogの確認

Dagster UIの「Catalog」メニューを開くと、dbtモデルがAssetsとして認識されていることが確認できました。

![Dagster Asset Catalog](/images/dagster-dbt-integration/dagster-dbt-catalog.png)

- **seeds**: raw_orders, raw_users
- **staging**: stg_orders, stg_users
- **marts**: user_order_summary

各Assetには「Postgres」「dbt」のタグが付与されており、dbtモデルであることが一目でわかります。

### Lineage（依存関係）の確認

「Lineage」メニューでは、dbtモデル間の依存関係がグラフで表示されます。

![Dagster Lineage](/images/dagster-dbt-integration/dagster-dbt-lineage.png)

```
raw_orders  →  stg_orders  →  user_order_summary
raw_users   →  stg_users   ↗
```

dbtの`ref()`関数で定義した依存関係が、Dagsterのリネージとして正しく反映されています。これはかなり便利ですね。

### Materializeの実行

「Jobs」→「dbt_job」→「Launchpad」→「Launch Run」でジョブを実行します。

実行後、「Runs」画面でログを確認すると、以下のような出力が見られました。

```
Finished dbt command: `dbt build --select fqn:*`.
Finished execution of step "my_dbt_assets" in 3.54s.
```

全5つのAssets（seeds 2 + models 3）が正常にマテリアライズされました。

---

## dbt testの連携

dbtのテスト機能もDagsterから実行できるか確認してみます。

### 1. schema.ymlの作成

stagingモデルにテストを定義します。

```yaml:dbt_project/models/staging/schema.yml
version: 2

models:
  - name: stg_users
    columns:
      - name: user_id
        tests:
          - unique
          - not_null

  - name: stg_orders
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: user_id
        tests:
          - not_null
          - relationships:
              arguments:
                to: ref('stg_users')
                field: user_id
```

`stg_users`には`user_id`のユニーク制約とNOT NULL制約、`stg_orders`には`order_id`と`user_id`の制約に加えて、`user_id`が`stg_users`に存在することを確認するリレーションシップテストを定義しました。

:::message
dbt 1.11以降では、`relationships`テストの引数は`arguments`でネストする形式が推奨されています。古い形式（直接`to`と`field`を指定）を使うと非推奨警告が表示されます。
:::

### 2. dbt test単体での確認

まずdbt単体でテストが動作することを確認します。

```bash
docker compose run --rm dagster bash -c "cd dbt_project && dbt test --profiles-dir ."
```

```
16:34:09  1 of 6 PASS not_null_stg_orders_order_id
16:34:09  2 of 6 PASS not_null_stg_orders_user_id
16:34:09  3 of 6 PASS not_null_stg_users_user_id
16:34:09  4 of 6 PASS relationships_stg_orders_user_id__user_id__ref_stg_users_
16:34:09  5 of 6 PASS unique_stg_orders_order_id
16:34:09  6 of 6 PASS unique_stg_users_user_id

16:34:09  Done. PASS=6 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=6
```

6件のテスト全てがPASSしました。

### 3. Dagsterからのテスト実行

Dagster UIで「Jobs」→「dbt_job」→「Launchpad」→「Launch Run」を実行します。

Run詳細画面のログで、テスト結果が確認できました。

![dbt test結果](/images/dagster-dbt-integration/dagster-dbt-test-result.png)

```
Check not_null_stg_orders_order_id succeeded for materialization of stg_orders.
  status: pass
  dagster_dbt/failed_row_count: 0
```

`@dbt_assets`で`dbt build`コマンドを使用しているため、モデルの実行後に自動的にテストも実行されます。テスト結果は「ASSET_CHECK」イベントとしてDagster UIに表示され、各テストの成功/失敗や実行時間が確認できます。

これは良いですね。dbtのテスト結果がDagster上で一元管理されるのは運用上便利そうです。

---

## 部分実行

データパイプラインでは、特定のモデルだけを再実行したいケースがよくあります。Dagster + dbtの連携で、これができるか確認してみました。

### UI上での選択実行

1. 「Catalog」または「Lineage」画面を開く
2. 実行したいAssetのチェックボックスを選択（例: `stg_orders`だけ）
3. 「Materialize selected」をクリック

Run詳細画面のログを確認すると、選択したモデルのみが実行されていることがわかりました。

![部分実行のRun](/images/dagster-dbt-integration/dagster-dbt-partial-run.png)

```
Finished dbt command: `dbt build --select dbt_project.staging.stg_orders`.
```

`dbt build --select`オプションで特定のモデルのみが実行されています。また、そのモデルに関連するテスト（`unique_stg_orders_order_id`、`not_null_stg_orders_order_id`等）も一緒に実行されます。

### 部分実行の活用シーン

- **特定モデルの修正後の再実行**: stagingモデルを修正した場合、そのモデルと下流のmartsモデルだけを再実行
- **データ品質問題の調査**: 特定のモデルでテストが失敗した場合、そのモデルだけを再実行してデバッグ
- **増分更新**: 大量データを扱う場合、変更があった部分だけを効率的に更新

---

## 検証結果まとめ

今回の検証で確認できたことをまとめます。

| 検証項目 | 結果 |
|---------|------|
| dbtモデルのAsset認識 | ✅ 問題なし |
| Lineage（依存関係）表示 | ✅ 問題なし |
| Materialize実行 | ✅ 問題なし |
| dbt testの連携 | ✅ 問題なし |
| 部分実行 | ✅ 問題なし |

### dagster-dbtを使ってみた所感

実際に検証してみて感じたメリットは以下の通りです。

1. **統一的なオーケストレーション**: dbtモデルを他のDagster Assetsと同じインターフェースで管理できる
2. **可視化**: Dagster UIでリネージやマテリアライゼーション履歴を確認できる
3. **柔軟な実行**: 部分実行やスケジュール実行など、Dagsterの機能をフル活用できる
4. **テスト統合**: dbt testの結果がDagster上で一元管理される

### dbt単体とDagster連携の比較

| 機能 | dbt単体（CLI） | Dagster + dbt |
|------|---------------|---------------|
| リネージ可視化 | dbt docs（静的HTML） | Dagster UI（動的・実行状態付き） |
| 依存関係の制御 | dbt内部のみ | Python処理や他DB処理を含めた全体制御 |
| 失敗時の通知 | 別途構築が必要 | Sensor等で容易に実装可能 |
| 部分実行 | CLI引数で指定 | UIから選択して実行 |
| 実行履歴の管理 | target/run_results.json | UIで一覧・詳細確認（メタデータも永続化） |

「なぜわざわざDagsterでdbtを包むのか？」という問いに対しては、リネージの統合、UI上での部分実行、テストの可視化あたりが大きなメリットになると感じました。

### 注意点

- Docker環境ではマルチプロセス実行でエラーが発生したため、`in_process_executor`を使う必要がありました（本番環境では別のエグゼキュータを検討）
- `dbt build`を使うとテストも自動実行されますが、runとtestを分けたい場合は`assets.py`でコマンドを変更する必要があります

dbtを使っていてオーケストレーションツールを検討している場合、dagster-dbtは良い選択肢になりそうです。

---

## 関連記事

Dagsterの基本的な使い方については、以下の入門シリーズで解説しています。

https://zenn.dev/toshiro3/articles/dagster-introduction

https://zenn.dev/toshiro3/articles/dagster-resource-schedule

https://zenn.dev/toshiro3/articles/dagster-partition

---

## 参考資料

- [Dagster公式ドキュメント - dagster-dbt](https://docs.dagster.io/integrations/dbt)
- [dbt公式ドキュメント](https://docs.getdbt.com/)
- [dagster-dbt PyPI](https://pypi.org/project/dagster-dbt/)


