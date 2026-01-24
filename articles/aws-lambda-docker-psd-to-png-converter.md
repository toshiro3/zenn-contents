---
title: "AWS Lambda + Docker + ImageMagickでPSD→PNG自動変換システムを構築する"
emoji: "🎨"
type: "tech"
topics: ["aws", "lambda", "docker", "imagemagick", "python"]
published: true
---

## はじめに

デザイナーから受け取ったPSDファイルをWeb用のPNGに変換する作業、手動でやっていませんか？

本記事では、S3にPSDファイルをアップロードするだけで自動的にPNGに変換されるサーバーレスシステムを構築します。

**この記事で構築するもの：**
- S3へのPSDアップロードをトリガーにLambdaが起動
- Docker + ImageMagick + WandでPSD→PNG変換
- 変換後のPNGを同じS3バケットに保存
- AWS CodeBuildによるCI/CDパイプライン

**対象読者：**
- AWSでサーバーレスアプリケーションを構築したい方
- Lambda + Dockerの実践例を知りたい方
- ImageMagickをLambdaで使いたい方

## アーキテクチャ

```
┌─────────────────┐     ┌────────────────────────────────────┐     ┌─────────────────┐
│   S3 (Input)    │     │     Lambda (Docker Container)      │     │   S3 (Output)   │
│                 │     │                                    │     │                 │
│  *.psd upload   │────▶│  Python 3.12 + ImageMagick + Wand  │────▶│  converted/*.png│
│                 │     │                                    │     │                 │
└─────────────────┘     └────────────────────────────────────┘     └─────────────────┘
```

S3イベント通知でLambdaをトリガーし、変換結果を同じバケットの`converted/`プレフィックス配下に出力します。

## 技術選定

### なぜDockerイメージ？

Lambda標準ランタイムにはImageMagickが含まれていません。Lambda Layerでバイナリを追加する方法もありますが、以下の理由からDockerイメージを選択しました。

| 方式 | メリット | デメリット |
|------|---------|-----------|
| Lambda Layer | デプロイが軽量 | 依存関係の管理が複雑、サイズ制限 |
| **Docker** | 環境の再現性、ローカルテスト容易 | イメージサイズが大きい |

特にImageMagickのような複雑な依存関係を持つライブラリは、Dockerで管理する方が圧倒的に楽です。

### なぜWand？

PythonからImageMagickを使う方法は主に2つあります。

**1. subprocess（コマンド実行）**
```python
import subprocess
subprocess.run(['convert', 'input.psd[0]', 'output.png'])
```

**2. Wand（Pythonバインディング）**
```python
from wand.image import Image
with Image(filename='input.psd[0]') as img:
    img.save(filename='output.png')
```

Wandを選んだ理由は以下の通りです。

- **Pythonic**: with文でリソース管理、例外処理が自然
- **メタデータアクセス**: `img.width`, `img.colorspace`などに直接アクセス
- **エラーハンドリング**: ImageMagickのエラーがPython例外として扱える

## 実装のポイント

### ロジック分離（テスタビリティ）

S3とのI/O処理と変換ロジックを分離することで、S3なしでもユニットテストが可能になります。

```
src/
├── app.py         # Lambda ハンドラー（S3 I/O層）
└── psd_logic.py   # 変換ロジック（純粋関数）
```

**psd_logic.py（抜粋）**
```python
@dataclass
class ConversionResult:
    success: bool
    input_path: str
    output_path: Optional[str]
    # ... その他のメタデータ

def convert_psd_to_png(
    input_path: str,
    output_path: str,
    options: ConversionOptions
) -> ConversionResult:
    """PSDをPNGに変換（S3に依存しない純粋関数）"""
    with Image(filename=f"{input_path}[0]") as img:
        # 変換処理
        img.format = 'png'
        img.save(filename=output_path)
```

**app.py（抜粋）**
```python
def lambda_handler(event, context):
    # S3からダウンロード
    s3.download_file(bucket, key, local_input)
    
    # 変換（psd_logicを呼び出し）
    result = convert_psd_to_png(local_input, local_output, options)
    
    # S3にアップロード
    s3.upload_file(local_output, output_bucket, output_key)
```

この設計により、テストコードはS3モックなしで動作します。

```python
def test_convert_basic(sample_psd, tmp_path):
    """変換ロジックのみをテスト（S3不要）"""
    output = tmp_path / "output.png"
    result = convert_psd_to_png(str(sample_psd), str(output), ConversionOptions())
    assert result.success
    assert output.exists()
```

### PSDの`[0]`指定

PSDファイルはマルチレイヤー構造を持ちます。ImageMagickでは`[0]`を指定することで「統合イメージ（Composite）」を取得できます。

```python
# [0]なし → 全レイヤーを個別に処理（意図しない結果に）
Image(filename='input.psd')

# [0]あり → 統合イメージのみ取得（通常はこちら）
Image(filename='input.psd[0]')
```

### CMYK対応

印刷用に作成されたPSDファイルはCMYKカラースペースの場合があります。これをそのままPNGにすると色がおかしくなります。

```python
if img.colorspace == 'cmyk':
    logger.warning(f"CMYK detected, converting to sRGB: {input_path}")
    img.transform_colorspace('srgb')
```

CMYKを検出したらsRGBに変換し、ログに警告を出力します。

:::message
**カラープロファイルについて**

`transform_colorspace('srgb')` は、元データにICCプロファイルが埋め込まれていない場合でも変換を試みます。ただし、プロファイル未埋め込みのCMYKファイルでは意図しない色化けが起きることがあります。

厳密な色再現が必要な場合は、`sRGB.icc`などのプロファイルファイルを同梱し、明示的に適用する方法もあります。本実装では実用上十分な「sRGBへの自動変換」を採用しています。
:::

### 無限ループ防止

S3イベントトリガーで注意すべきは「無限ループ」です。変換後のPNGをS3にアップロードすると、それがまたトリガーになってしまう可能性があります。

対策として、以下の2重チェックを行っています。

**1. S3イベントフィルター（template.yaml）**
```yaml
Events:
  S3PsdUpload:
    Type: S3
    Properties:
      Bucket: !Ref InputBucket
      Events: s3:ObjectCreated:*
      Filter:
        S3Key:
          Rules:
            - Name: suffix
              Value: .psd  # PSDファイルのみ
```

**2. Lambdaコード内チェック**
```python
if key.startswith(OUTPUT_PREFIX):
    logger.info(f"Skipping output file: {key}")
    return
```

### マルチステージDockerビルド

本番イメージからテストコードを除外するため、マルチステージビルドを採用しています。

```dockerfile
# Stage 1: base - 共通基盤
FROM public.ecr.aws/lambda/python:3.12 AS base
RUN dnf install -y ImageMagick ImageMagick-devel && dnf clean all
COPY requirements.txt ${LAMBDA_TASK_ROOT}/
RUN pip install --no-cache-dir -r requirements.txt

# Stage 2: test - テスト環境
FROM base AS test
COPY requirements-dev.txt ${LAMBDA_TASK_ROOT}/
RUN pip install --no-cache-dir -r requirements-dev.txt
COPY src/ ${LAMBDA_TASK_ROOT}/
COPY tests/ ${LAMBDA_TASK_ROOT}/tests/
ENTRYPOINT ["python", "-m"]
CMD ["pytest", "tests/", "-v"]

# Stage 3: production - 本番環境
FROM base AS production
COPY src/ ${LAMBDA_TASK_ROOT}/
CMD ["app.lambda_handler"]
```

テスト実行は`--target test`、本番ビルドは`--target production`で切り替えます。

## デプロイ手順

ソースコードは以下からダウンロードできます。

https://github.com/toshiro3/lambda-psd-converter-demo

### 前提条件

- AWS CLI v2（認証設定済み）
- Docker
- 約10分の時間

### Step 1: ソースコードの準備

```bash
# ダウンロード＆展開
unzip lambda-psd-converter-demo.zip
cd lambda-psd-converter-demo
```

### Step 2: ローカルテスト（オプション）

```bash
# Dockerイメージビルド＆テスト実行
make test
```

14件のテストがパスすればOKです。

### Step 3: ソースコード用S3バケット作成

```bash
# 変数設定
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="psd-converter-source-${ACCOUNT_ID}"
INPUT_BUCKET="psd-input-${ACCOUNT_ID}"

# バケット作成
aws s3 mb s3://${SOURCE_BUCKET}
```

### Step 4: ソースコードをアップロード

```bash
zip -r lambda-psd-converter-demo.zip . \
  -x ".git/*" -x "test-output/*" -x ".aws-sam/*" -x "*.zip"
aws s3 cp lambda-psd-converter-demo.zip s3://${SOURCE_BUCKET}/
```

### Step 5: CodeBuildプロジェクト作成

```bash
aws cloudformation deploy \
  --template-file codebuild-stack.yaml \
  --stack-name psd-converter-codebuild \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    SourceBucketName=${SOURCE_BUCKET} \
    InputBucketName=${INPUT_BUCKET}
```

### Step 6: ビルド実行

```bash
aws codebuild start-build --project-name psd-converter-build
```

ビルドには約3〜5分かかります。進捗はCloudWatchログで確認できます。

```bash
aws logs tail /aws/codebuild/psd-converter-build --follow
```

### Step 7: 動作確認

テスト用のPSDファイルを生成してアップロードします。

```bash
# テスト用PSD生成（Dockerを利用）
docker build --target production -t psd-converter:latest .
docker run --rm \
  -v "$(pwd):/work" \
  --entrypoint convert \
  psd-converter:latest \
  -size 500x500 xc:red -fill blue -draw "circle 250,250 250,100" /work/sample.psd

# S3にアップロード
aws s3 cp sample.psd s3://${INPUT_BUCKET}/

# 変換結果を確認（数秒待つ）
sleep 5
aws s3 ls s3://${INPUT_BUCKET}/converted/
```

`converted/sample.png`が表示されれば成功です！

### クリーンアップ

検証が終わったら、以下のコマンドでリソースを削除します。

```bash
# S3バケットの中身を削除
aws s3api delete-objects \
  --bucket ${INPUT_BUCKET} \
  --delete "$(aws s3api list-object-versions \
    --bucket ${INPUT_BUCKET} \
    --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' \
    --output json)"

# スタック削除
aws cloudformation delete-stack --stack-name psd-converter-dev
aws cloudformation wait stack-delete-complete --stack-name psd-converter-dev

aws cloudformation delete-stack --stack-name psd-converter-codebuild
aws cloudformation wait stack-delete-complete --stack-name psd-converter-codebuild

# ECR削除
aws ecr delete-repository --repository-name psd-converter --force

# ソースバケット削除
aws s3 rm s3://${SOURCE_BUCKET} --recursive
aws s3 rb s3://${SOURCE_BUCKET}
```

## パフォーマンスとコスト

### 実行結果

| 項目 | 値 |
|------|-----|
| 入力サイズ | 4MB (500x500 PSD) |
| 出力サイズ | 190KB (PNG) |
| 変換時間 | 約800ms |
| コールドスタート | 約4.5秒 |
| メモリ使用量 | 120MB / 2048MB |

:::message
**コールドスタートについて**

Dockerイメージを使用したLambdaのコールドスタートは4〜5秒程度が一般的です。VPC内で動作させる場合は、ENIアタッチの時間が加わりさらに遅くなる可能性があります。

処理頻度が低く、かつ即時性が求められる場合は「Provisioned Concurrency」の利用を検討してください。ただし、追加コストが発生するため、要件に応じて判断が必要です。
:::

### コスト見積もり（月間1,000ファイル処理の場合）

| リソース | 計算 | 月額 |
|---------|------|------|
| Lambda | 1000回 × 5秒 × 2GB | 約$0.17 |
| S3 | 1000 PUT + 1000 GET | 約$0.01 |
| ECR | 500MB | 約$0.05 |
| **合計** | | **約$0.23** |

ほぼ無料で運用できます。

## トラブルシューティング

### ImageMagick policy エラー

```
wand.exceptions.PolicyError: not allowed by the security policy
```

ImageMagickはセキュリティ上の理由から、デフォルトでPSD/PSフォーマットの処理やリソース使用量を制限しています。Dockerfileで`policy.xml`を修正して制限を解除しています。

```dockerfile
# ImageMagick の policy.xml を修正
RUN POLICY_FILE=$(find /etc -name "policy.xml" 2>/dev/null | head -1) && \
    if [ -n "$POLICY_FILE" ]; then \
        # PSD/PS の読み書きを許可
        sed -i 's/<policy domain="coder" rights="none" pattern="PS" \/>/<policy domain="coder" rights="read|write" pattern="PS" \/>/g' "$POLICY_FILE" && \
        sed -i 's/<policy domain="coder" rights="none" pattern="PSD" \/>/<policy domain="coder" rights="read|write" pattern="PSD" \/>/g' "$POLICY_FILE" && \
        # メモリ制限を緩和（大きな PSD 対応）
        sed -i 's/<policy domain="resource" name="memory" value="[^"]*"\/>/<policy domain="resource" name="memory" value="2GiB"\/>/g' "$POLICY_FILE" && \
        sed -i 's/<policy domain="resource" name="map" value="[^"]*"\/>/<policy domain="resource" name="map" value="4GiB"\/>/g' "$POLICY_FILE" && \
        sed -i 's/<policy domain="resource" name="disk" value="[^"]*"\/>/<policy domain="resource" name="disk" value="8GiB"\/>/g' "$POLICY_FILE"; \
    fi
```

設定は`make check-imagemagick`で確認できます。

### メモリ不足 / タイムアウト / ストレージ不足

大きなPSDファイル（100MB以上）を処理する場合は、以下の点に注意してください。

**エフェメラルストレージ（/tmp）について**

Lambdaの`/tmp`領域はデフォルトで**512MB**です。最近の複雑なPSD（スマートオブジェクトや高解像度データを含むもの）は数GBに達することがあり、`s3.download_file`の時点でディスク容量不足になる可能性があります。

`template.yaml`で以下を調整してください。

```yaml
Globals:
  Function:
    Timeout: 300       # 最大900秒
    MemorySize: 4096   # 最大10240MB
    EphemeralStorage:
      Size: 5120       # 最大10240MB（デフォルト512MB）
```

| PSDサイズ | 推奨メモリ | 推奨/tmp |
|----------|-----------|----------|
| ~10MB | 1024MB | 512MB |
| ~50MB | 2048MB | 1024MB |
| ~100MB | 3072MB | 2048MB |
| 100MB~ | 4096MB+ | 5120MB+ |

## まとめ

本記事では、AWS Lambda + Docker + ImageMagick（Wand）を使ったPSD→PNG自動変換システムを構築しました。

**ポイント：**
- Lambda + Dockerで複雑な依存関係を管理
- Wandで Pythonic な画像処理
- ロジック分離でテスタビリティ向上
- CodeBuildでCI/CDパイプライン構築

このアーキテクチャは、PSD以外にも様々なフォーマット変換に応用できます。ぜひ試してみてください！

## 参考

- [AWS Lambda コンテナイメージサポート](https://docs.aws.amazon.com/lambda/latest/dg/images-create.html)
- [Wand Documentation](https://docs.wand-py.org/)
- [ImageMagick Security Policy](https://imagemagick.org/script/security-policy.php)
