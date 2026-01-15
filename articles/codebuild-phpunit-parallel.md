---
title: "CodeBuild Batch で PHPUnit を並列実行できるか検証してみた"
emoji: "🧪"
type: "tech"
topics: ["aws", "codebuild", "phpunit", "laravel", "ci"]
published: false
---

## はじめに

業務でユニットテストの実行時間が課題になっていたため、AWS CodeBuild の Batch ビルド機能で並列実行できないか検証しました。

結論から言うと、PHPUnit でも並列実行は可能ですが、セットアップコストの問題があり、テスト時間が短い場合はメリットが出にくいという結果でした。

## やりたかったこと

CodeBuild には `build-fanout` という機能があり、`codebuild-tests-run` CLI と組み合わせることで、テストファイルを動的に分割して並列実行できます。

```yaml
batch:
  build-fanout:
    parallelism: 3  # 3並列で実行
```

```bash
codebuild-tests-run \
  --test-command "jest" \
  --files-search "codebuild-glob-search 'tests/**/*.test.js'"
```

Jest や pytest ではこの方式でうまく動くようです。

## つまずいたポイント

### PHPUnit は複数ファイルを引数に取れない

`codebuild-tests-run` はテストファイルのリストを返しますが、PHPUnit は複数ファイルを引数に取れません。

```bash
# Jest / pytest ならこれでOK
jest tests/UserTest.js tests/OrderTest.js
pytest tests/UserTest.py tests/OrderTest.py

# PHPUnit はこれができない
vendor/bin/phpunit tests/UserServiceTest.php tests/OrderServiceTest.php
# → 最初のファイルしか実行されない
```

Pest も PHPUnit ベースのため、同じ制約があります。

## 回避策

### 方式1: build-list + testsuite（静的分割）

`phpunit.xml` に testsuite を事前定義し、`build-list` で各シャードに割り当てる方式です。

```xml
<!-- phpunit.xml -->
<testsuites>
    <testsuite name="shard-1">
        <file>tests/Unit/UserServiceTest.php</file>
        <file>tests/Unit/OrderServiceTest.php</file>
    </testsuite>
    <testsuite name="shard-2">
        <file>tests/Unit/PaymentServiceTest.php</file>
        <file>tests/Unit/NotificationServiceTest.php</file>
    </testsuite>
    <testsuite name="shard-3">
        <file>tests/Unit/InventoryServiceTest.php</file>
        <file>tests/Unit/ReportServiceTest.php</file>
    </testsuite>
</testsuites>
```

```yaml
# buildspec-parallel.yml
batch:
  build-list:
    - identifier: shard_1
      env:
        variables:
          SHARD_NUM: "1"
    - identifier: shard_2
      env:
        variables:
          SHARD_NUM: "2"
    - identifier: shard_3
      env:
        variables:
          SHARD_NUM: "3"

phases:
  build:
    commands:
      - vendor/bin/phpunit --testsuite "shard-${SHARD_NUM}"
```

シンプルですが、テストファイル追加時に `phpunit.xml` のメンテナンスが必要です。

### 方式2: build-fanout + ラッパースクリプト（動的分割）

ラッパースクリプトで動的に `phpunit.xml` を生成する方式です。

```xml
<!-- phpunit.template.xml -->
<phpunit>
    <testsuites>
        <testsuite name="CodeBuild Parallel Test Suite">
            {{TEST_FILES}}
        </testsuite>
    </testsuites>
</phpunit>
```

```bash
#!/bin/bash
# run_parallel_phpunit.sh

FILES=$@

# ファイルリストを <file>path</file> 形式に変換
FORMATTED_FILES=$(echo "$FILES" | sed 's|^|<file>|; s| |</file><file>|g; s|$|</file>|')

# テンプレートからphpunit.xmlを生成
sed "s@{{TEST_FILES}}@$FORMATTED_FILES@g" phpunit.template.xml > phpunit.dynamic.xml

# PHPUnit実行
./vendor/bin/phpunit -c phpunit.dynamic.xml
```

```yaml
# buildspec-fanout.yml
batch:
  build-fanout:
    parallelism: 3

phases:
  build:
    commands:
      - |
        codebuild-tests-run \
          --test-command "./run_parallel_phpunit.sh" \
          --files-search "codebuild-glob-search 'tests/**/*Test.php'"
```

この方式なら `parallelism` を変えるだけでシャード数を調整でき、testsuite のメンテナンスも不要です。

## 検証結果

約43秒かかるテストスイート（6ファイル、20テスト）で検証しました。

| 方式 | Duration | 備考 |
|-----|----------|------|
| 直列実行 | 約1分 | ベースライン |
| build-list（静的分割） | 約3分 | 3シャード |
| build-fanout（動的分割） | 約2分 | 3シャード |

### なぜ並列実行の方が遅いのか

各シャードが独立したビルドとして実行されるため、`composer install` がシャードの数だけ走ります。

```
┌─────────────────────────────────────────────────────────────┐
│ 直列実行                                                      │
│                                                              │
│ [セットアップ 12秒][    テスト 43秒    ]                        │
│ ├──────────────────────────────────────┤                     │
│              合計: 約1分                                      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 並列実行（3シャード）                                          │
│                                                              │
│ シャード1: [セットアップ 12秒][テスト 14秒]                      │
│ シャード2: [セットアップ 12秒][テスト 14秒]                      │
│ シャード3: [セットアップ 12秒][テスト 15秒]                      │
│            ├─────────────────────────────┤                   │
│              合計: 約2〜3分（オーケストレーション含む）            │
└─────────────────────────────────────────────────────────────┘
```

テスト自体は並列で14〜15秒に短縮されますが、セットアップコストが3倍になるため、全体の Duration は直列実行より長くなりました。

**損益分岐点**: セットアップに12秒かかる場合、テストが36秒以上短縮されないと並列化のメリットが出ません。今回のケースでは、テスト短縮分（43秒 → 15秒 = 28秒短縮）がセットアップ増加分（12秒 × 2 = 24秒増加）をわずかに上回る程度で、オーケストレーションのオーバーヘッドを含めると逆転してしまいました。

### セットアップコストの最適化

今回の検証では `composer install` に約12秒かかっていましたが、以下の方法で短縮できる可能性があります。

- **CodeBuild のキャッシュ機能**: `vendor/` ディレクトリをキャッシュすれば、2回目以降のインストール時間を大幅に短縮できます
- **カスタムイメージの利用**: 依存関係をプリインストールしたDockerイメージを使えば、セットアップをほぼゼロにできます

キャッシュを最適化してセットアップを数秒まで削れれば、損益分岐点は5分程度まで下がるかもしれません。

### build-fanout と build-list の差

build-list が約3分、build-fanout が約2分と差が出ました。2回検証して同じ傾向だったため、build-fanout の方がオーバーヘッドが小さい可能性があります。

推測ですが、`build-list` は各ビルドの定義を個別にパースして管理するのに対し、`build-fanout` は同一の設定をコピーして展開するため、オーケストレーション側のプロビジョニングが効率的なのかもしれません。

## 代替案: Paratest との比較

1マシン内で並列実行する [paratest](https://github.com/paratestphp/paratest) も選択肢になります。

```bash
composer require --dev brianium/paratest
vendor/bin/paratest -p 4 --testsuite Unit
```

| 方式 | 特徴 |
|-----|------|
| **CodeBuild Batch** | 複数マシンで実行。スケーラビリティは高いが、マシン起動のオーバーヘッドがある |
| **Paratest** | 1マシン内で実行。CPU数に依存するが、起動が速くセットアップは1回で済む |

テストが10分程度であれば Paratest で十分なケースが多いと思います。テストが30分以上かかる、または CPU 数以上の並列度が必要な場合は CodeBuild Batch の検討価値があります。

## まとめ

CodeBuild Batch で PHPUnit を並列実行することは可能ですが、以下の点で注意が必要です。

- PHPUnit は複数ファイルを引数に取れないため、ラッパースクリプトが必要
- 各シャードでセットアップが走るため、短いテストでは逆効果
- キャッシュやカスタムイメージでセットアップを最適化すれば、損益分岐点を下げられる
- テストが10分程度なら Paratest、30分以上なら CodeBuild Batch を検討

Jest や pytest など、複数ファイルを引数に取れるフレームワークであれば、もう少しスムーズに導入できると思います。

## 検証用リポジトリ

https://github.com/toshiro3/codebuild-parallel-test-demo
