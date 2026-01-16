---
title: "PHPUnit のテスト結果を CodeBuild と Allure Report で可視化してみた"
emoji: "📊"
type: "tech"
topics: ["aws", "codebuild", "phpunit", "allure", "ci"]
published: false
---

## はじめに

CI/CD でテストを実行しているものの、結果をログで確認するだけになっていませんか？

テスト結果を可視化することで、失敗したテストの特定や、テスト実行時間の推移を把握しやすくなります。

この記事では、AWS CodeBuild でテスト結果を可視化する2つの方法を比較検証しました。

- **CodeBuild 標準レポート機能**: AWS コンソールで確認、セットアップが簡単
- **Allure Report**: リッチな UI、詳細なテスト管理

### 忙しい人のための比較まとめ

| 項目 | CodeBuild 標準 | Allure Report |
|-----|---------------|---------------|
| セットアップ難易度 | ◎ 簡単 | △ やや複雑 |
| レポート保持 | 30日（自動削除） | S3 に保存（設定次第） |
| 履歴・トレンド表示 | 簡易的（成功率・実行時間の推移） | 詳細（最大20件） |
| UI の詳細度 | 基本的 | リッチ |
| 費用 | 無料（ビルド料金に含まれる）※ | S3 費用のみ |
| 並列実行時の結合 | 別々に表示 | 自動で統合 |

※ 30日を超えて保存するために S3 エクスポートを有効にした場合、別途 S3 のストレージ料金が発生します（[料金ページ](https://aws.amazon.com/codebuild/pricing/)）。

**結論**: まずは CodeBuild 標準レポートから始めて、必要に応じて Allure を併用するのがおすすめです。

## 検証環境

- PHP 8.2
- PHPUnit 10
- allure-framework/allure-phpunit v3

検証用リポジトリ: https://github.com/toshiro3/codebuild-report-demo

## 方法1: CodeBuild 標準レポート機能

### 設定方法

buildspec に `reports` セクションを追加するだけです。

```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      php: 8.2
    commands:
      - composer install --no-interaction --prefer-dist

  build:
    commands:
      - mkdir -p reports
      - vendor/bin/phpunit --testsuite Unit --log-junit reports/junit.xml

reports:
  phpunit-reports:
    files:
      - "reports/junit.xml"
    file-format: JUNITXML
```

### 必要な IAM 権限

レポートグループの作成に追加の権限が必要です。

```json
{
  "Effect": "Allow",
  "Action": [
    "codebuild:CreateReportGroup",
    "codebuild:CreateReport",
    "codebuild:UpdateReport",
    "codebuild:BatchPutTestCases"
  ],
  "Resource": "*"
}
```

### 確認方法

CodeBuild コンソール → プロジェクト → ビルド履歴 → 「レポート」タブ

テスト件数、成功/失敗、実行時間などが確認できます。

失敗したテストケースを選択すると、その場ですぐにスタックトレースを確認できるため、膨大なビルドログからエラー箇所を探し出すストレスから解放されます。

### 特徴

| 項目 | 内容 |
|-----|------|
| セットアップ | 簡単（buildspec に数行追加） |
| 保持期間 | 30日（自動削除） |
| 費用 | 無料（ビルド料金に含まれる） |
| 履歴・トレンド | 簡易的（成功率・実行時間の推移） |

30日で自動削除されるため、それ以上保持したい場合は S3 へのエクスポートが必要です（[公式ドキュメント](https://docs.aws.amazon.com/codebuild/latest/userguide/test-reporting.html)）。

レポートグループの画面では、過去のレポート実行結果がグラフ化されて表示されます。ただし、Allure のような詳細なステップや傾向の分析機能は限定的です。

## 方法2: Allure Report

### Allure とは

Allure は多くのテストフレームワークに対応したレポート生成ツールです。リッチな UI で、履歴・トレンド表示にも対応しています。

### セットアップ

#### 1. composer で allure-phpunit をインストール

```json
{
  "require-dev": {
    "phpunit/phpunit": "^10.0",
    "allure-framework/allure-phpunit": "^3"
  }
}
```

#### 2. phpunit.xml に Extension を追加

```xml
<extensions>
    <bootstrap class="Qameta\Allure\PHPUnit\AllureExtension">
        <parameter name="config" value="config/allure.config.php"/>
    </bootstrap>
</extensions>
```

#### 3. 設定ファイルを作成

```php
<?php
// config/allure.config.php

return [
    'outputDirectory' => 'allure-results',
];
```

#### 4. buildspec でレポート生成・S3 アップロード

```yaml
version: 0.2

env:
  variables:
    ALLURE_BUCKET: "your-allure-report-bucket"

phases:
  install:
    runtime-versions:
      php: 8.2
    commands:
      - composer install --no-interaction --prefer-dist
      - curl -o allure-2.24.0.tgz -Ls https://repo.maven.apache.org/maven2/io/qameta/allure/allure-commandline/2.24.0/allure-commandline-2.24.0.tgz
      - tar -zxvf allure-2.24.0.tgz
      - export PATH="$PWD/allure-2.24.0/bin:$PATH"

  pre_build:
    commands:
      # 前回の履歴をダウンロード
      - mkdir -p allure-results/history
      - aws s3 sync "s3://${ALLURE_BUCKET}/history" ./allure-results/history || true

  build:
    commands:
      - vendor/bin/phpunit --testsuite Unit

  post_build:
    commands:
      - export PATH="$PWD/allure-2.24.0/bin:$PATH"
      # レポート生成
      - allure generate ./allure-results -o ./allure-report --clean
      # S3 にアップロード
      - aws s3 sync ./allure-report "s3://${ALLURE_BUCKET}/report" --delete
      - aws s3 sync ./allure-report/history "s3://${ALLURE_BUCKET}/history"
```

### S3 バケットの設定

静的ウェブサイトホスティングを有効化します。

:::message alert
テスト結果にはクラス名やエラーログに含まれる環境変数など、機密情報が混じる可能性があります。社内プロジェクトで利用する場合は、CloudFront + OAC（Origin Access Control）と WAF や IP 制限を組み合わせる、あるいは VPN 内からのみアクセス可能な設定を推奨します。
:::

:::details S3 バケット設定用コマンド
```bash
ALLURE_BUCKET="your-allure-report-bucket"

# バケット作成
aws s3 mb "s3://${ALLURE_BUCKET}"

# 静的ウェブサイトホスティングを有効化
aws s3 website "s3://${ALLURE_BUCKET}" --index-document index.html

# パブリックアクセス設定（必要に応じて）
aws s3api put-public-access-block \
  --bucket "${ALLURE_BUCKET}" \
  --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

# バケットポリシー
cat > bucket-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::${ALLURE_BUCKET}/report/*"
    }
  ]
}
EOF

aws s3api put-bucket-policy --bucket "${ALLURE_BUCKET}" --policy file://bucket-policy.json
```
:::

### 確認方法

```
http://<bucket-name>.s3-website-ap-northeast-1.amazonaws.com/report/index.html
```

### Allure アトリビュートでレポートを充実

テストコードにアトリビュートを追加すると、レポートがより見やすくなります。

```php
<?php

use PHPUnit\Framework\TestCase;
use Qameta\Allure\Attribute\DisplayName;
use Qameta\Allure\Attribute\Description;
use Qameta\Allure\Attribute\Severity;

#[DisplayName('ユーザーサービスのテスト')]
#[Description('ユーザー関連の機能をテストします')]
class UserServiceTest extends TestCase
{
    #[DisplayName('ユーザー名のバリデーション - 正常系')]
    #[Severity(Severity::CRITICAL)]
    public function testValidUsername(): void
    {
        // ...
    }
}
```

### 履歴・トレンド表示

Allure の特徴は履歴機能です。複数回ビルドを実行すると、TREND グラフに過去の実行結果が蓄積されていきます。

ポイントは `pre_build` で前回の履歴をダウンロードし、`post_build` で新しい履歴をアップロードすることです。

```yaml
pre_build:
  commands:
    - aws s3 sync "s3://${ALLURE_BUCKET}/history" ./allure-results/history || true

post_build:
  commands:
    - aws s3 sync ./allure-report/history "s3://${ALLURE_BUCKET}/history"
```

Allure は `history` ディレクトリ内の JSON ファイルを読み取ってトレンドを生成します。この設定では前回の1回分だけでなく、蓄積された履歴データを同期しています。現状の最大20件程度であれば問題ありませんが、運用が長期化する場合は古いデータのパージを検討してください。

### 特徴

| 項目 | 内容 |
|-----|------|
| セットアップ | やや複雑（S3、IAM 設定が必要） |
| レポート保持 | S3 に保存（設定次第で長期保存も可能） |
| 履歴・トレンド | 最大20件 |
| 費用 | S3 ストレージ費用（微量） |

## 比較の詳細

### 履歴表示期間の注意点

Allure の履歴は **最大20件** に制限されています（[公式ドキュメント](https://allurereport.org/docs/history-and-retries/)）。

| 実行頻度 | CodeBuild 標準 | Allure Report |
|---------|---------------|---------------|
| 1日1回 | 30日分 | 20日分 |
| 1日2回 | 30日分 | 10日分 |
| 1日3回 | 30日分 | 約7日分 |

実行頻度が高い場合、履歴の表示期間は CodeBuild 標準の方が長くなる場合があります。

なお、Allure はビルドごとにディレクトリを分けて S3 に保存する設定にすれば、過去のレポートを長期保存することも可能です。

## 用途別おすすめ

### CodeBuild 標準レポートがおすすめ

- まずは手軽にテスト結果を可視化したい
- 追加のインフラ管理を避けたい
- AWS コンソールから過去のビルド履歴とともに確認したい

### Allure Report がおすすめ

- よりリッチな UI でテスト結果を確認したい
- 並列実行したテスト結果を1つにまとめたい
- テストにアトリビュート（DisplayName、Severity など）を付けて詳細に管理したい

## おわりに

CodeBuild 標準レポート機能は手軽に導入でき、日々の開発には十分な機能があります。

よりリッチな可視化が必要な場合は Allure Report も選択肢になります。セットアップに手間はかかりますが、詳細な UI やアトリビュートによるテスト管理ができます。

まずは CodeBuild 標準レポートから始めて、必要に応じて Allure を併用するのが良いアプローチだと思います。

## 検証用リポジトリ

https://github.com/toshiro3/codebuild-report-demo
