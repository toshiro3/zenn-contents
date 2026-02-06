---
title: "Polars入門 ── 基本操作からpandas・DuckDBとの使い分けまで検証した"
emoji: "🐻‍❄️"
type: "tech"
topics: ["polars", "python", "pandas", "duckdb", "dataengineering"]
published: true
---

## はじめに

データエンジニアリングの現場で、DataFrameライブラリの選択肢が増えています。長年デファクトだった **pandas** に加え、**DuckDB** や **Polars** といったRust/C++製の高速ライブラリが台頭してきました。

Polarsは「速い」「モダン」と評判を聞くものの、実際に触って確かめないとわからないことも多いので、今回は基本操作から遅延評価の最適化の中身、pandas・DuckDBとの具体的な違いまでをひと通り検証してみました。

### この記事でやったこと

- uv + Python 3.14 でモダンな検証環境を構築
- Polars の基本操作（select / filter / sort / group_by）を試す
- CSV / Parquet の読み書きと `scan`（遅延読み込み）の挙動を確認
- 遅延評価（Lazy API）で `explain()` によるクエリプランの最適化を確認
- pandas と同じ操作を並べて書き方・挙動の違いを比較
- DuckDB との相互運用を試し、使い分けの観点を整理

### 検証リポジトリ

今回の検証コードは以下にまとめています。各ステップ（step1.py〜step6.py）を順に実行すれば、この記事と同じ結果を再現できます。

https://github.com/toshiro3/polars-intro

### 検証環境

| 項目 | バージョン |
|------|-----------|
| Python | 3.14.2 |
| Polars | 1.38.0 |
| pandas | 3.0.0 |
| DuckDB | 1.4.4 |
| PyArrow | 23.0.0 |
| uv | 最新 |
| OS | macOS |

### Polarsとは

Polars は **Rust製** の高速DataFrameライブラリです。公式サイトやドキュメントから特徴をまとめると、以下のようになります。

- **Rust製**: コアがRustで実装されており、メモリ効率とパフォーマンスに優れる
- **列指向（Columnar）**: Apache Arrow のメモリモデルを採用し、SIMD演算やキャッシュ効率が高い
- **遅延評価（Lazy Evaluation）**: クエリオプティマイザがプランを最適化してから実行する
- **依存関係ゼロ**: NumPy や PyArrow に依存せず、単体で動作する
- **Expressionベース**: `pl.col()` を使った直感的なAPIで、操作の意図が明確になる

公式ベンチマーク（PDS-H）では、pandasに対して **30倍以上** のパフォーマンスを記録しているとのことです。この記事では速度よりもAPIの使い勝手や挙動の違いを中心に検証しました。

## 1. 環境構築（uv + Python 3.14）

今回はパッケージマネージャに **uv** を選びました。uvもRust製なので、Polarsとの組み合わせは「Rust × Rust」のモダンなPython環境になります。

### uv のインストール

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### プロジェクト作成

```bash
mkdir polars-intro && cd polars-intro
uv init --python 3.14
uv add polars
```

実行結果:

```
Using CPython 3.14.2
Creating virtual environment at: .venv
Resolved 3 packages in 142ms
Prepared 2 packages in 1.02s
Installed 2 packages in 8ms
 + polars==1.38.0
 + polars-runtime-32==1.38.0
```

ここで気づいたのが、`polars` 本体と `polars-runtime-32` の2パッケージがインストールされる点です。調べてみると、v1.37から導入されたアーキテクチャで、`polars` がpure Pythonのインターフェース、`polars-runtime-32` がRust製バイナリを含むランタイムという分離構成になっていました。

もう1つ注目したのは、`uv add polars` だけで完結していること。NumPyやPyArrowなどの追加依存はありません。この「依存関係ゼロ」は後で pandas と比較したときにはっきり差が出ました。

### 動作確認

uvの環境では `python` コマンドを直接叩いても `command not found` になります。`uv run` 経由で実行する必要があります。

```bash
uv run python --version
# Python 3.14.2
```

## 2. 基本操作

### DataFrameの作成と表示

まずは小さなDataFrameを作って表示してみました。

```python
import polars as pl

df = pl.DataFrame({
    "name": ["Alice", "Bob", "Charlie", "Diana"],
    "age": [28, 35, 42, 31],
    "department": ["Engineering", "Sales", "Engineering", "Sales"],
    "salary": [600000, 450000, 750000, 500000],
})

print(df)
```

```
shape: (4, 4)
┌─────────┬─────┬─────────────┬────────┐
│ name    ┆ age ┆ department  ┆ salary │
│ ---     ┆ --- ┆ ---         ┆ ---    │
│ str     ┆ i64 ┆ str         ┆ i64    │
╞═════════╪═════╪═════════════╪════════╡
│ Alice   ┆ 28  ┆ Engineering ┆ 600000 │
│ Bob     ┆ 35  ┆ Sales       ┆ 450000 │
│ Charlie ┆ 42  ┆ Engineering ┆ 750000 │
│ Diana   ┆ 31  ┆ Sales       ┆ 500000 │
└─────────┴─────┴─────────────┴────────┘
```

最初に目を引いたのが、**各列のヘッダ直下に型情報**（`str`, `i64`）が表示されること。pandasでは `dtypes` を別途確認する必要がありますが、Polarsはprintするだけで型がわかります。

型推論の結果も確認してみました。

```python
print(df.schema)
# Schema({'name': String, 'age': Int64, 'department': String, 'salary': Int64})
```

整数は `Int64`、文字列は `String` に推論されました。pandasの `int64` / `object` と比べると、Polarsの型名はより直感的です。

### select / filter / sort / with_columns

Polarsの操作は **Expression API** を中心に構成されています。`pl.col()` で列を指定し、メソッドチェーンで変換を記述するスタイルです。以下のデータで各操作を試しました。

```python
df = pl.DataFrame({
    "name": ["Alice", "Bob", "Charlie", "Diana", "Eve"],
    "age": [28, 35, 42, 31, 26],
    "department": ["Engineering", "Sales", "Engineering", "Sales", "Engineering"],
    "salary": [600000, 450000, 750000, 500000, 550000],
})
```

**列の選択（select）**

```python
# シンプルな列選択
df.select("name", "age")

# Expressionで加工しながら選択
df.select(
    pl.col("name"),
    (pl.col("salary") / 10000).alias("salary_man"),
)
```

```
shape: (5, 2)
┌─────────┬────────────┐
│ name    ┆ salary_man │
│ ---     ┆ ---        │
│ str     ┆ f64        │
╞═════════╪════════════╡
│ Alice   ┆ 60.0       │
│ Bob     ┆ 45.0       │
│ Charlie ┆ 75.0       │
│ Diana   ┆ 50.0       │
│ Eve     ┆ 55.0       │
└─────────┴────────────┘
```

`i64 / 10000` の演算で自動的に `f64` に型変換されていました。`alias()` で列名を指定でき、pandasの `rename` に相当する操作が Expression の中で完結するのは書きやすいです。

**行のフィルタ（filter）**

```python
df.filter(
    (pl.col("department") == "Engineering") & (pl.col("age") >= 30)
)
```

```
shape: (1, 4)
┌─────────┬─────┬─────────────┬────────┐
│ name    ┆ age ┆ department  ┆ salary │
│ ---     ┆ --- ┆ ---         ┆ ---    │
│ str     ┆ i64 ┆ str         ┆ i64    │
╞═════════╪═════╪═════════════╪════════╡
│ Charlie ┆ 42  ┆ Engineering ┆ 750000 │
└─────────┴─────┴─────────────┴────────┘
```

複数条件の結合には `&`（AND）、`|`（OR）を使い、各条件を `()` で囲みます。この書き方自体はpandasと同じですが、`pl.col()` を使う分、どの列への操作かが明確になる印象です。

**ソート（sort）**

```python
df.sort("department", "salary", descending=[False, True])
```

```
shape: (5, 4)
┌─────────┬─────┬─────────────┬────────┐
│ name    ┆ age ┆ department  ┆ salary │
│ ---     ┆ --- ┆ ---         ┆ ---    │
│ str     ┆ i64 ┆ str         ┆ i64    │
╞═════════╪═════╪═════════════╪════════╡
│ Charlie ┆ 42  ┆ Engineering ┆ 750000 │
│ Alice   ┆ 28  ┆ Engineering ┆ 600000 │
│ Eve     ┆ 26  ┆ Engineering ┆ 550000 │
│ Diana   ┆ 31  ┆ Sales       ┆ 500000 │
│ Bob     ┆ 35  ┆ Sales       ┆ 450000 │
└─────────┴─────┴─────────────┴────────┘
```

`descending` にリストを渡すことで、列ごとにソート方向を指定できました。

**列の追加（with_columns）**

```python
df.with_columns(
    (pl.col("salary") * 12).alias("annual_salary"),
    (pl.col("age") >= 30).alias("is_senior"),
)
```

```
shape: (5, 6)
┌─────────┬─────┬─────────────┬────────┬───────────────┬───────────┐
│ name    ┆ age ┆ department  ┆ salary ┆ annual_salary ┆ is_senior │
│ ---     ┆ --- ┆ ---         ┆ ---    ┆ ---           ┆ ---       │
│ str     ┆ i64 ┆ str         ┆ i64    ┆ i64           ┆ bool      │
╞═════════╪═════╪═════════════╪════════╪═══════════════╪═══════════╡
│ Alice   ┆ 28  ┆ Engineering ┆ 600000 ┆ 7200000       ┆ false     │
│ Bob     ┆ 35  ┆ Sales       ┆ 450000 ┆ 5400000       ┆ true      │
│ Charlie ┆ 42  ┆ Engineering ┆ 750000 ┆ 9000000       ┆ true      │
│ Diana   ┆ 31  ┆ Sales       ┆ 500000 ┆ 6000000       ┆ true      │
│ Eve     ┆ 26  ┆ Engineering ┆ 550000 ┆ 6600000       ┆ false     │
└─────────┴─────┴─────────────┴────────┴───────────────┴───────────┘
```

ここで重要なのは、`with_columns` が**新しいDataFrameを返す**非破壊的な操作だという点です。pandasの `df["new_col"] = ...` は元のDataFrameを直接変更しますが、Polarsでは元の `df` はそのまま残ります。また、比較演算の結果がネイティブな `bool` 型になっているのも確認できました。

### group_by + agg

```python
df.group_by("department").agg(
    pl.col("salary").mean().alias("avg_salary"),
    pl.col("salary").max().alias("max_salary"),
    pl.len().alias("headcount"),
)
```

```
shape: (2, 4)
┌─────────────┬───────────────┬────────────┬───────────┐
│ department  ┆ avg_salary    ┆ max_salary ┆ headcount │
│ ---         ┆ ---           ┆ ---        ┆ ---       │
│ str         ┆ f64           ┆ i64        ┆ u32       │
╞═════════════╪═══════════════╪════════════╪═══════════╡
│ Sales       ┆ 475000.0      ┆ 500000     ┆ 2         │
│ Engineering ┆ 633333.333333 ┆ 750000     ┆ 3         │
└─────────────┴───────────────┴────────────┴───────────┘
```

ここで `pl.len()` を使いましたが、Polarsには `count()` もあります。使い分けとしては、**行数を数えるなら `pl.len()`、特定列の非NULL値を数えるなら `pl.col("col").count()`** です。Polars v1.x 以降では行数カウントに `pl.len()` が推奨されています。

なお、戻り値が `u32`（符号なし32ビット整数）になっている点にも注目です。pandasでは `int64` になるので、他システムとの連携時に型の違いが効いてくる場面がありそうです。

もう1点、`group_by` の結果は**行の順序が保証されない**ことも確認しました。確定した順序が必要な場合は `.sort()` を付ける必要があります。

### メソッドチェーン

一連の操作をメソッドチェーンで繋げてみました。

```python
result = (
    df
    .filter(pl.col("department") == "Engineering")
    .with_columns(
        (pl.col("salary") * 1.1).cast(pl.Int64).alias("salary_after_raise")
    )
    .select("name", "salary", "salary_after_raise")
    .sort("salary_after_raise", descending=True)
)
```

```
shape: (3, 3)
┌─────────┬────────┬────────────────────┐
│ name    ┆ salary ┆ salary_after_raise │
│ ---     ┆ ---    ┆ ---                │
│ str     ┆ i64    ┆ i64                │
╞═════════╪════════╪════════════════════╡
│ Charlie ┆ 750000 ┆ 825000             │
│ Alice   ┆ 600000 ┆ 660000             │
│ Eve     ┆ 550000 ┆ 605000             │
└─────────┴────────┴────────────────────┘
```

filter → with_columns → select → sort と、上から下に読み下せるのは実際に書いてみると気持ちがいいです。`.cast(pl.Int64)` で明示的に型を指定できるのも、暗黙の型変換に頼らない設計として好印象でした。

## 3. CSV / Parquet の読み書き

データエンジニアリングで頻出する CSV と Parquet の読み書きを検証しました。

### 書き出し

まず10,000行のサンプルデータを生成し、CSV と Parquet で書き出しました。

```python
import polars as pl

df = pl.DataFrame({
    "user_id": range(1, 10001),
    "name": [f"user_{i:05d}" for i in range(1, 10001)],
    "age": [20 + (i % 50) for i in range(10000)],
    "department": [["Engineering", "Sales", "Marketing", "HR", "Finance"][i % 5]
                   for i in range(10000)],
    "salary": [300000 + (i * 100 % 500000) for i in range(10000)],
})

df.write_csv("sample.csv")
df.write_parquet("sample.parquet")
```

### 読み込み（read vs scan）

Polarsのファイル読み込みには `read` と `scan` の2つのアプローチがあります。

| メソッド | 戻り値 | 挙動 |
|---------|--------|------|
| `read_csv` / `read_parquet` | `DataFrame` | 即座に全データをメモリに読み込む |
| `scan_csv` / `scan_parquet` | `LazyFrame` | `collect()` まで読み込みを遅延する |

```python
# 即時読み込み → DataFrame
df_csv = pl.read_csv("sample.csv")
df_pq = pl.read_parquet("sample.parquet")

# 遅延読み込み → LazyFrame
lf_csv = pl.scan_csv("sample.csv")
lf_pq = pl.scan_parquet("sample.parquet")

print(type(lf_pq))
# <class 'polars.lazyframe.frame.LazyFrame'>
```

`scan` を使うと、戻り値が `LazyFrame` になり、実際のファイル読み込みは `collect()` を呼ぶまで実行されません。この遅延の間にPolarsのクエリオプティマイザが最適化を行ってくれます。

### scan → filter → collect

`scan` を使った基本的なパターンを試しました。

```python
result = (
    pl.scan_parquet("sample.parquet")
    .filter(pl.col("department") == "Engineering")
    .filter(pl.col("salary") >= 500000)
    .select("user_id", "name", "salary")
    .sort("salary", descending=True)
    .head(5)
    .collect()
)
```

```
shape: (5, 3)
┌─────────┬────────────┬────────┐
│ user_id ┆ name       ┆ salary │
│ ---     ┆ ---        ┆ ---    │
│ i64     ┆ str        ┆ i64    │
╞═════════╪════════════╪════════╡
│ 9996    ┆ user_09996 ┆ 799500 │
│ 4996    ┆ user_04996 ┆ 799500 │
│ 4991    ┆ user_04991 ┆ 799000 │
│ 9991    ┆ user_09991 ┆ 799000 │
│ 9986    ┆ user_09986 ┆ 798500 │
└─────────┴────────────┴────────┘
```

この scan → filter → collect の流れが、Polarsで大規模データを扱う際の基本パターンになります。Parquet の場合、必要な列だけ読む「プロジェクションプッシュダウン」や、条件に合う行だけ読む「プレディケートプッシュダウン」が効くので、メモリ効率が大幅に改善されるはずです。この最適化の中身は次のセクションで `explain()` を使って確認しました。

### ファイルサイズ比較

CSV と Parquet のファイルサイズも比較してみました。

```python
import os

csv_size = os.path.getsize("sample.csv")
pq_size = os.path.getsize("sample.parquet")
print(f"CSV size:     {csv_size:>10,} bytes")
print(f"Parquet size: {pq_size:>10,} bytes")
print(f"Compression ratio: {csv_size / pq_size:.1f}x")
```

```
CSV size:        336,929 bytes
Parquet size:     42,604 bytes
Compression ratio: 7.9x
```

10,000行のデータで約7.9倍の差がつきました。Parquetは列指向フォーマットかつ内部で圧縮がかかるため、特に同じ値が繰り返される列（今回の `department` のように5種類しかないカテゴリ列）で高い圧縮率を発揮するようです。

## 4. 遅延評価（Lazy API）の深掘り

Polarsの遅延評価は単に「処理を後回しにする」だけではなく、**クエリオプティマイザ**がプランを分析し、最適な実行順序に書き換えてくれるとのこと。`explain()` でその最適化の中身を実際に確認してみました。

### 論理プラン vs 最適化済みプラン

以下のクエリに対して、最適化前後のプランを比較します。

```python
lazy_query = (
    pl.scan_parquet("sample.parquet")
    .filter(pl.col("department") == "Engineering")
    .filter(pl.col("salary") >= 500000)
    .select("user_id", "name", "salary")
    .sort("salary", descending=True)
    .head(5)
)
```

**論理プラン（自分が書いた通りの操作順序）**

```python
print(lazy_query.explain(optimized=False))
```

```
SLICE[offset: 0, len: 5]
  SORT BY [descending: [true]] [col("salary")]
    SELECT [col("user_id"), col("name"), col("salary")]
      FILTER [(col("salary")) >= (500000)]
      FROM
        FILTER [(col("department")) == ("Engineering")]
        FROM
          Parquet SCAN [sample.parquet]
          PROJECT */5 COLUMNS
          ESTIMATED ROWS: 10000
```

**最適化済みプラン（Polarsが書き換えた実行順序）**

```python
print(lazy_query.explain(optimized=True))
```

```
SORT BY [slice: (0, 5), descending: [true]] [col("salary")]
  simple π 3/3 ["user_id", "name", "salary"]
    Parquet SCAN [sample.parquet]
    PROJECT 4/5 COLUMNS
    SELECTION: [([(col("salary")) >= (500000)]) & ([(col("department")) == ("Engineering")])]
    ESTIMATED ROWS: 10000
```

### 確認できた最適化

2つのプランを比較すると、3つの最適化が行われていました。

**① フィルタの結合（Predicate Pushdown）**

論理プランでは2つのFILTERが個別に並んでいました。

```
FILTER salary >= 500000
  FROM
    FILTER department == "Engineering"
```

最適化後は1つのSELECTIONに統合され、しかもParquetスキャンの段階まで押し下げられていました。

```
SELECTION: [(salary >= 500000) & (department == "Engineering")]
```

**② プロジェクションプッシュダウン**

論理プランでは全5列を読み込む `PROJECT */5 COLUMNS` だったのが、最適化後は `PROJECT 4/5 COLUMNS` に変わっていました。`select` で指定していない `age` 列をParquet読み込みの時点でスキップしています。`department` はフィルタ条件で必要なため、4列読み込みとなっていました。

**③ SORT + SLICE の融合**

論理プランでは全件ソートしてから先頭5件を切り出す SORT → SLICE の2ステップでした。最適化後は `SORT BY [slice: (0, 5)]` と融合されており、全件ソートせずにTop-5だけを効率的に取得する形になっていました。

このように、ユーザーが書いたコードの順序をオプティマイザが賢く書き換えてくれることが、`explain()` の出力で実際に確認できました。

### DataFrame → LazyFrame の変換

`read_csv` で読み込んだ後でも `.lazy()` で LazyFrame に変換できることも確認しました。

```python
df = pl.read_csv("sample.csv")
result = (
    df.lazy()
    .filter(pl.col("department").is_in(["Engineering", "Marketing"]))
    .group_by("department")
    .agg(
        pl.col("salary").mean().alias("avg_salary"),
        pl.col("salary").median().alias("median_salary"),
        pl.len().alias("count"),
    )
    .sort("avg_salary", descending=True)
    .collect()
)
```

```
shape: (2, 4)
┌─────────────┬────────────┬───────────────┬───────┐
│ department  ┆ avg_salary ┆ median_salary ┆ count │
│ ---         ┆ ---        ┆ ---           ┆ ---   │
│ str         ┆ f64        ┆ f64           ┆ u32   │
╞═════════════╪════════════╪═══════════════╪═══════╡
│ Marketing   ┆ 549950.0   ┆ 549950.0      ┆ 2000  │
│ Engineering ┆ 549750.0   ┆ 549750.0      ┆ 2000  │
└─────────────┴────────────┴───────────────┴───────┘
```

つまり、既存の Eager コードを Lazy に移行する際は `.lazy()` と `.collect()` で挟むだけで済みます。段階的に導入できるのはありがたい設計です。

## 5. pandasとの比較

「pandasとどう違うのか」は自分自身が一番気になっていた点なので、同じ操作を並べて比較してみました。

### 事前準備

```bash
uv add pandas
```

```
Resolved 8 packages in 518ms
Installed 4 packages in 52ms
 + numpy==2.4.2
 + pandas==3.0.0
 + python-dateutil==2.9.0.post0
 + six==1.17.0
```

この時点で依存関係の差がはっきり出ました。`polars` は追加パッケージ1つ（runtime）で済んだのに対し、`pandas` は numpy を含む4パッケージが必要でした。

### フィルタ → 列選択 → ソート

```python
# pandas
result_pd = (
    df_pd[df_pd["department"] == "Engineering"]
    [["user_id", "name", "salary"]]
    .sort_values("salary", ascending=False)
    .head(5)
)

# Polars
result_pl = (
    df_pl
    .filter(pl.col("department") == "Engineering")
    .select("user_id", "name", "salary")
    .sort("salary", descending=True)
    .head(5)
)
```

pandasはブラケット記法 `df[condition][columns]` が中心で、列選択とフィルタが同じ `[]` 記号で行われます。Polarsは `.filter()` と `.select()` で操作が明確に分離されているので、コードの意図が読み取りやすいと感じました。

### group_by + 集計

```python
# pandas
result_pd = (
    df_pd
    .groupby("department")
    .agg(
        avg_salary=("salary", "mean"),
        max_salary=("salary", "max"),
        headcount=("user_id", "count"),
    )
    .sort_values("avg_salary", ascending=False)
    .reset_index()
)

# Polars
result_pl = (
    df_pl
    .group_by("department")
    .agg(
        pl.col("salary").mean().alias("avg_salary"),
        pl.col("salary").max().alias("max_salary"),
        pl.col("user_id").count().alias("headcount"),  # 非NULLカウント
    )
    .sort("avg_salary", descending=True)
)
```

pandasは `("列名", "関数名")` のタプルを渡すスタイル。Polarsは `pl.col("salary").mean().alias("avg_salary")` と、Expression APIで「何をどう集計するか」が自己文書化される書き方です。ここでは pandas の `count` に合わせて `pl.col().count()` を使いましたが、単純な行数カウントであれば前述の通り `pl.len()` を使う方がPolarsでは一般的です。

また、pandasの `groupby` 後はグループキーがインデックスに入るため `.reset_index()` が必要ですが、Polarsにはインデックスの概念がないのでそもそも不要です。この「インデックスがない」設計は、使ってみると思った以上にシンプルで快適でした。

### 列の追加

```python
# pandas（破壊的）
df_pd["annual_salary"] = df_pd["salary"] * 12

# Polars（非破壊的）
df_pl.with_columns(
    (pl.col("salary") * 12).alias("annual_salary"),
)
```

pandasは元のDataFrameを直接変更します。Polarsは新しいDataFrameを返すため、元データが意図せず変更されるリスクがありません。

### NULLの扱い ── 最も大きな違い

個人的に今回の検証で一番インパクトがあったのがこれです。

```python
# pandas
df_null_pd = pd.DataFrame({
    "name": ["Alice", "Bob", "Charlie"],
    "score": [90.0, None, 85.0],
})
print(df_null_pd.dtypes)
# name      str
# score    float64   ← Noneを入れるとfloat64に化ける
```

```python
# Polars
df_null_pl = pl.DataFrame({
    "name": ["Alice", "Bob", "Charlie"],
    "score": [90, None, 85],
})
print(df_null_pl.schema)
# Schema({'name': String, 'score': Int64})   ← Int64のままnullを保持
```

```
[Polars]
shape: (3, 2)
┌─────────┬───────┐
│ name    ┆ score │
│ ---     ┆ ---   │
│ str     ┆ i64   │
╞═════════╪═══════╡
│ Alice   ┆ 90    │
│ Bob     ┆ null  │
│ Charlie ┆ 85    │
└─────────┴───────┘
```

pandasでは整数列に `None` が1つでも含まれると、列全体が `float64` に変換されてしまいます。Polarsは `null` を型とは独立したビットマスクで管理するため、`Int64` のまま欠損値を保持できていました。

null判定と埋め込みもExpression APIで統一的に書けます。

```python
df_null_pl.with_columns(
    pl.col("score").is_null().alias("is_null"),
    pl.col("score").fill_null(0).alias("score_filled"),
)
```

```
shape: (3, 4)
┌─────────┬───────┬─────────┬──────────────┐
│ name    ┆ score ┆ is_null ┆ score_filled │
│ ---     ┆ ---   ┆ ---     ┆ ---          │
│ str     ┆ i64   ┆ bool    ┆ i64          │
╞═════════╪═══════╪═════════╪══════════════╡
│ Alice   ┆ 90    ┆ false   ┆ 90           │
│ Bob     ┆ null  ┆ true    ┆ 0            │
│ Charlie ┆ 85    ┆ false   ┆ 85           │
└─────────┴───────┴─────────┴──────────────┘
```

`fill_null(0)` の結果も `i64` のまま。型が一切ブレません。データパイプラインの信頼性を考えると、この差は大きいです。

## 6. DuckDBとの使い分け

DuckDBもデータエンジニアリングで人気のツールなので、Polarsとの相互運用を試してみました。

### 事前準備

```bash
uv add duckdb pyarrow
```

なお、最初は `pyarrow` なしで試したところ、DuckDBからPolars DataFrameを直接クエリする際に `ModuleNotFoundError: No module named 'pyarrow'` が発生しました。Parquetファイル経由であれば PyArrow 不要ですが、メモリ上のDataFrameを渡す場合は Arrow フォーマット経由のデータ受け渡しが必要なため、PyArrow が必須になります。

### 同じ操作をSQLとDataFrame APIで

```python
import duckdb

# DuckDB（SQL）
result_duck = duckdb.sql("""
    SELECT
        department,
        AVG(salary) AS avg_salary,
        MAX(salary) AS max_salary,
        COUNT(*) AS headcount
    FROM 'sample.parquet'
    WHERE salary >= 400000
    GROUP BY department
    ORDER BY avg_salary DESC
""")

# Polars（DataFrame API）
result_pl = (
    pl.scan_parquet("sample.parquet")
    .filter(pl.col("salary") >= 400000)
    .group_by("department")
    .agg(
        pl.col("salary").mean().alias("avg_salary"),
        pl.col("salary").max().alias("max_salary"),
        pl.len().alias("headcount"),
    )
    .sort("avg_salary", descending=True)
    .collect()
)
```

どちらも同じ結果が返ってきました。SQLに慣れていればDuckDB、DataFrame操作で書きたければPolarsという使い分けが自然にできそうです。

### DuckDB → Polars の相互運用

DuckDBはPolars DataFrameを**直接SQLでクエリ**できることを確認しました。

```python
df_pl = pl.read_parquet("sample.parquet")

result = duckdb.sql("""
    SELECT department, AVG(salary) AS avg_salary
    FROM df_pl
    WHERE age >= 30
    GROUP BY department
    ORDER BY avg_salary DESC
""")
```

```
┌─────────────┬────────────┐
│ department  │ avg_salary │
│   varchar   │   double   │
├─────────────┼────────────┤
│ Finance     │   550650.0 │
│ HR          │   550550.0 │
│ Marketing   │   550450.0 │
│ Sales       │   550350.0 │
│ Engineering │   550250.0 │
└─────────────┴────────────┘
```

Python変数の `df_pl` をそのままSQLの `FROM` 句に書けるのは便利です。

逆方向の変換も試しました。DuckDBの結果を `.pl()` でPolars DataFrameに変換できます。

```python
result_as_pl = duckdb.sql("""
    SELECT department, COUNT(*) AS cnt
    FROM 'sample.parquet'
    GROUP BY department
    ORDER BY cnt DESC
""").pl()

print(type(result_as_pl))
# <class 'polars.dataframe.frame.DataFrame'>
```

### Polars自身のSQL機能

Polars自体にもSQLインターフェースがあることがわかったので、こちらも試しました。

```python
df = pl.read_parquet("sample.parquet")
result_sql = pl.sql("""
    SELECT
        department,
        AVG(salary) AS avg_salary,
        COUNT(*) AS headcount
    FROM df
    WHERE salary >= 500000
    GROUP BY department
    ORDER BY avg_salary DESC
""")
print(type(result_sql))
# <class 'polars.lazyframe.frame.LazyFrame'>
```

興味深かったのは、`pl.sql()` の戻り値が **LazyFrame** だった点です。SQLで書いても遅延評価の恩恵（クエリ最適化）を受けられます。

### 使い分けの指針

検証を通じて感じた使い分けの観点をまとめます。

| 観点 | Polars | DuckDB |
|------|--------|--------|
| インターフェース | DataFrame API（Expression） | SQL |
| 得意な場面 | ETLパイプライン、データ変換 | アドホック分析、探索的クエリ |
| 遅延評価 | LazyFrame で明示的に制御 | 常に遅延評価 |
| ファイル直接クエリ | `scan_parquet` 等 | `FROM 'file.parquet'` |
| 相互運用 | `.to_pandas()` 等 | `.pl()` で Polars に変換可能 |
| 依存関係 | ゼロ | ゼロ |

実務では、**DuckDBでアドホックにデータを探索** → **Polarsでパイプラインを構築**という流れが相性が良さそうだと感じました。両者はParquetファイルを介してスムーズに連携できます。

## まとめ

### 検証を通じて感じたPolarsの強み

- **型の信頼性**: nullでも型が壊れない点が、pandasとの最も大きな違いだった
- **操作の明確さ**: Expression API + メソッドチェーンで、コードが自己文書化される
- **依存関係ゼロ**: pip install だけで即使える軽量さ。pandasとの差が明確だった
- **クエリ最適化**: `explain()` で確認した通り、オプティマイザが賢くプランを書き換えてくれる

### Polarsが向いていると感じた場面

- 数十万〜数億行規模のデータ処理
- ETLパイプラインの構築（型安全性と非破壊操作が重要な場面）
- Parquet を中心としたデータレイク/ウェアハウスとの連携
- pandasのメモリ不足や速度に不満がある場合

特に `scan_parquet` + Lazy API の組み合わせは、必要な列と行だけをメモリに載せるため、実メモリを超えるサイズのデータファイルに対してもストリーミング実行で処理できる可能性があります。今回の検証は1万行の小規模データでしたが、大規模データでこそPolarsの遅延評価とプッシュダウン最適化が本領を発揮するはずです。

### pandas / DuckDB との使い分け

| 場面 | おすすめ |
|------|---------|
| 既存のpandasコードがある、小規模データ | pandas |
| SQLで素早くデータを探索したい | DuckDB |
| ETLパイプラインを型安全に構築したい | Polars |
| 大規模Parquetデータの集計・変換 | Polars or DuckDB |
| Jupyter でインタラクティブに分析 | pandas or DuckDB |

今回の検証を通じて、これらは「どれか1つを選ぶ」ものではなく、場面に応じて使い分ける（あるいは組み合わせる）ものだと実感しました。Polarsを選択肢に加えることで、データエンジニアリングの引き出しが確実に広がると思います。
