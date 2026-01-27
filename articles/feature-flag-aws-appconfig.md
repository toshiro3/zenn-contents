---
title: "AWS AppConfigでFeature Flagを試してみた【Laravel連携】"
emoji: "🏴"
type: "tech"
topics: ["aws", "laravel", "php", "featureflag", "appconfig"]
published: false
---

## はじめに

Feature Flagツール比較検証の第3弾として、**AWS AppConfig**を検証しました。

これまでの検証記事：
- [Laravel Pennantで機能フラグを試してみた](https://zenn.dev/toshiro3/articles/feature-flag-laravel-pennant)
- [Unleash OSSでFeature Flagを試してみた【Laravel連携】](https://zenn.dev/toshiro3/articles/feature-flag-unleash)

### AWS AppConfigとは

AWS AppConfigは、AWS Systems Managerの一部として提供される設定管理サービスです。Feature Flagの管理機能も備えています。

**主な特徴：**
- AWSマネージドサービス（インフラ管理不要）
- 段階的デプロイ（Bake time、ロールバック対応）
- 環境別管理（Application × Environment構成）
- IAMによるアクセス制御
- Feature Flags型とFreeform型の2種類の設定形式

## 検証環境

| 項目 | バージョン |
|------|-----------|
| PHP | 8.3 |
| Laravel | 12.x |
| AWS SDK for PHP | 3.x |
| Docker | 27.x |
| AWS CLI | 2.x |

### リポジトリ

検証用のソースコードはこちら：
https://github.com/toshiro3/appconfig-feature-flag-demo

## AppConfigの構成要素

AppConfigでFeature Flagを使うには、以下のリソースを作成します。

```
Application（アプリケーション）
  └── Environment（環境：dev / staging / production）
  └── Configuration Profile（設定プロファイル）
        └── Configuration Version（設定バージョン）

Deployment Strategy（デプロイ戦略）
```

**各リソースの役割：**

| リソース | 説明 |
|----------|------|
| Application | プロジェクト単位のコンテナ |
| Environment | dev/staging/prodなど環境を分離 |
| Configuration Profile | 設定の種類（Feature Flags型 or Freeform型） |
| Deployment Strategy | デプロイ方法（即時 or 段階的） |

### アーキテクチャ概要

```
┌─────────────────────────────────────────────────────────────────────┐
│                        管理面（設定の更新）                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   AWS CLI / Console                                                 │
│         │                                                           │
│         ▼                                                           │
│   ┌─────────────────────────────────────────────┐                  │
│   │  AWS AppConfig                               │                  │
│   │  ├── Application                             │                  │
│   │  │   ├── Environment (dev/staging/prod)     │                  │
│   │  │   └── Configuration Profile              │                  │
│   │  │       └── Hosted Configuration (JSON)    │                  │
│   │  └── Deployment Strategy                    │                  │
│   └─────────────────────────────────────────────┘                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                        実行面（設定の取得）                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Laravel App (ECS/EC2/Local)                                       │
│         │                                                           │
│         │ IAM Role / Credentials                                    │
│         ▼                                                           │
│   ┌─────────────────┐    ┌─────────────────────────┐              │
│   │ AWS SDK for PHP │───▶│ AppConfig Data API      │              │
│   │ (AppConfigData) │    │ - StartConfigurationSession             │
│   └─────────────────┘    │ - GetLatestConfiguration │              │
│         │                └─────────────────────────┘              │
│         ▼                                                           │
│   ┌─────────────────┐                                              │
│   │ Cache (Redis/   │                                              │
│   │ File/Memory)    │                                              │
│   └─────────────────┘                                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Step 1: 環境構築（Docker + AWS CLI）

### ディレクトリ構成

```
appconfig-feature-flag-demo/
├── compose.yml
├── Dockerfile
├── src/                          # Laravelプロジェクト
├── appconfig/                    # AppConfig関連ファイル
│   ├── flags/                    # フラグ定義JSON
│   │   ├── v1-basic.json
│   │   ├── v2-targeting.json
│   │   ├── v3-rollout.json
│   │   └── staging.json
│   └── scripts/                  # セットアップ用スクリプト
│       ├── 01-create-resources.sh
│       ├── 02-deploy-flags.sh
│       └── cleanup.sh
└── .gitignore
```

### Dockerfile

```dockerfile
FROM php:8.3-cli

RUN apt-get update && apt-get install -y \
    git \
    unzip \
    libzip-dev \
    && docker-php-ext-install zip

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

EXPOSE 8000

CMD ["php", "artisan", "serve", "--host=0.0.0.0", "--port=8000"]
```

### compose.yml

```yaml
services:
  app:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ./src:/var/www/html
      - ~/.aws:/root/.aws:ro  # AWS認証情報をマウント（読み取り専用）
    environment:
      - AWS_PROFILE=${AWS_PROFILE:-default}
      - AWS_DEFAULT_REGION=ap-northeast-1
    working_dir: /var/www/html
```

ポイントは `~/.aws:/root/.aws:ro` でホストのAWS認証情報をコンテナにマウントしている点です。これにより、AWS CLIと同じプロファイルがコンテナ内のAWS SDKでも使えます。

### セットアップ

```bash
# プロジェクトディレクトリ作成
mkdir appconfig-feature-flag-demo && cd appconfig-feature-flag-demo

# Laravelプロジェクト作成
docker compose run --rm app composer create-project laravel/laravel .

# AWS SDK for PHPをインストール
docker compose run --rm app composer require aws/aws-sdk-php

# コンテナ起動
docker compose up -d
```

## Step 2: AppConfigリソース作成（AWS CLI）

### AWSリソース作成スクリプト

**appconfig/scripts/01-create-resources.sh**:

```bash
#!/bin/bash
set -e

REGION="ap-northeast-1"
APP_NAME="feature-flag-demo"
ENV_NAME="development"
PROFILE_NAME="feature-flags"

echo "=== Creating AppConfig Resources ==="

# Application作成
APP_RESULT=$(aws appconfig create-application \
  --name "$APP_NAME" \
  --description "Feature flag verification for Laravel" \
  --region $REGION \
  --output json)
APP_ID=$(echo $APP_RESULT | jq -r '.Id')
echo "Application ID: $APP_ID"

# Environment作成
ENV_RESULT=$(aws appconfig create-environment \
  --application-id $APP_ID \
  --name "$ENV_NAME" \
  --description "Development environment" \
  --region $REGION \
  --output json)
ENV_ID=$(echo $ENV_RESULT | jq -r '.Id')
echo "Environment ID: $ENV_ID"

# Configuration Profile作成（Feature Flags型）
PROFILE_RESULT=$(aws appconfig create-configuration-profile \
  --application-id $APP_ID \
  --name "$PROFILE_NAME" \
  --location-uri "hosted" \
  --type "AWS.AppConfig.FeatureFlags" \
  --region $REGION \
  --output json)
PROFILE_ID=$(echo $PROFILE_RESULT | jq -r '.Id')
echo "Configuration Profile ID: $PROFILE_ID"

# 結果をファイルに保存
cat << EOF > appconfig/.env.appconfig
APPCONFIG_APPLICATION_ID=$APP_ID
APPCONFIG_ENVIRONMENT_ID=$ENV_ID
APPCONFIG_CONFIGURATION_PROFILE_ID=$PROFILE_ID
EOF

echo "=== Complete ==="
echo "Resource IDs saved to appconfig/.env.appconfig"
```

:::message
**Feature Flags型 vs Freeform型**

AppConfigには「Feature Flags型」と「Freeform型」の2種類のConfiguration Profileがあります。

- **Feature Flags型**: AWSコンソールのUIでフラグのON/OFF、カスタム属性を視覚的に管理できる。CLIの場合は `--content-type "application/vnd.amazonaws.appconfig.profiles+json"` の指定が**必須**
- **Freeform型**: 任意のJSON構造を保存可能。`--content-type "application/json"` で運用

機能フラグ用途では**Feature Flags型が推奨**です。本検証でも最終的にFeature Flags型でカスタム属性が正しく動作することを確認しました。
:::

### デプロイ用スクリプト

**appconfig/scripts/02-deploy-flags.sh**:

```bash
#!/bin/bash
set -e

REGION="ap-northeast-1"
STRATEGY_ID="<your-instant-strategy-id>"  # 後述

if [ -z "$1" ]; then
  echo "Usage: $0 <flags-json-file>"
  exit 1
fi

FLAGS_FILE=$1
source appconfig/.env.appconfig

echo "=== Deploying Feature Flags ==="

# Configuration Versionをアップロード（Feature Flags型用のcontent-type）
VERSION_RESULT=$(aws appconfig create-hosted-configuration-version \
  --application-id $APPCONFIG_APPLICATION_ID \
  --configuration-profile-id $APPCONFIG_CONFIGURATION_PROFILE_ID \
  --content-type "application/vnd.amazonaws.appconfig.profiles+json" \
  --content fileb://$FLAGS_FILE \
  --region $REGION \
  /dev/null)

LATEST_VERSION=$(echo "$VERSION_RESULT" | jq -r '.VersionNumber')
echo "Created version: $LATEST_VERSION"

# デプロイ実行
aws appconfig start-deployment \
  --application-id $APPCONFIG_APPLICATION_ID \
  --environment-id $APPCONFIG_ENVIRONMENT_ID \
  --configuration-profile-id $APPCONFIG_CONFIGURATION_PROFILE_ID \
  --configuration-version $LATEST_VERSION \
  --deployment-strategy-id $STRATEGY_ID \
  --region $REGION \
  --output json > /dev/null

echo "=== Deployment Complete ==="
```

:::message alert
**content-typeの指定が重要です**

Feature Flags型の場合、`--content-type "application/vnd.amazonaws.appconfig.profiles+json"` を指定しないと、カスタム属性がAPIレスポンスに含まれません。
:::

### 即時デプロイ用ストラテジー作成

デフォルトの `AppConfig.AllAtOnce` は `FinalBakeTimeInMinutes: 10` が設定されており、デプロイ完了まで10分かかります。検証用に即時デプロイのストラテジーを作成します。

```bash
aws appconfig create-deployment-strategy \
  --name "Instant" \
  --deployment-duration-in-minutes 0 \
  --growth-factor 100 \
  --final-bake-time-in-minutes 0 \
  --replicate-to NONE \
  --region ap-northeast-1
```

出力された `Id` を `02-deploy-flags.sh` の `STRATEGY_ID` に設定してください。

## Step 3: Laravel連携

### 設定ファイル

**src/config/appconfig.php**:

```php
<?php

return [
    'application_id' => env('APPCONFIG_APPLICATION_ID'),
    'environment_id' => env('APPCONFIG_ENVIRONMENT_ID'),
    'configuration_profile_id' => env('APPCONFIG_CONFIGURATION_PROFILE_ID'),
    'region' => env('AWS_DEFAULT_REGION', 'ap-northeast-1'),
];
```

### 環境変数

**src/.env** に追記：

```dotenv
APPCONFIG_APPLICATION_ID=<your-app-id>
APPCONFIG_ENVIRONMENT_ID=<your-env-id>
APPCONFIG_CONFIGURATION_PROFILE_ID=<your-profile-id>
AWS_PROFILE=<your-profile>
AWS_DEFAULT_REGION=ap-northeast-1
```

### FeatureFlagService

**src/app/Services/FeatureFlagService.php**:

```php
<?php

namespace App\Services;

use Aws\AppConfigData\AppConfigDataClient;
use Aws\Credentials\CredentialProvider;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Log;

class FeatureFlagService
{
    private AppConfigDataClient $client;
    private ?string $sessionToken = null;
    private array $flags = [];

    public function __construct()
    {
        $this->client = new AppConfigDataClient([
            'region' => config('appconfig.region'),
            'version' => 'latest',
            'credentials' => CredentialProvider::defaultProvider(),
        ]);
    }

    public function isEnabled(string $flagKey, array $context = []): bool
    {
        $flags = $this->getFlags();

        if (!isset($flags[$flagKey])) {
            return false;
        }

        $flagValue = $flags[$flagKey];
        $baseEnabled = $flagValue['enabled'] ?? false;

        // ユーザーターゲティング（allowed_usersが空でない場合のみ）
        if (!empty($flagValue['allowed_users']) && isset($context['user_id'])) {
            // Feature Flags型はカンマ区切り文字列、Freeform型は配列の可能性
            // 型安全のためキャストを行う
            $allowedUsers = is_array($flagValue['allowed_users'])
                ? $flagValue['allowed_users']
                : explode(',', (string)($flagValue['allowed_users'] ?? ''));
            return in_array($context['user_id'], $allowedUsers, true);
        }

        // パーセンテージロールアウト
        if (isset($flagValue['rollout_percentage']) && isset($context['user_id'])) {
            return $this->isInRollout($context['user_id'], $flagKey, (int)$flagValue['rollout_percentage']);
        }

        return $baseEnabled;
    }

    public function getFlags(): array
    {
        if (!empty($this->flags)) {
            return $this->flags;
        }

        return Cache::remember('appconfig_flags', 30, function () {
            return $this->fetchFlags();
        });
    }

    private function fetchFlags(): array
    {
        try {
            $session = $this->client->startConfigurationSession([
                'ApplicationIdentifier' => config('appconfig.application_id'),
                'EnvironmentIdentifier' => config('appconfig.environment_id'),
                'ConfigurationProfileIdentifier' => config('appconfig.configuration_profile_id'),
            ]);

            $this->sessionToken = $session['InitialConfigurationToken'];

            $config = $this->client->getLatestConfiguration([
                'ConfigurationToken' => $this->sessionToken,
            ]);

            $content = $config['Configuration']->getContents();

            if (empty($content)) {
                return [];
            }

            return json_decode($content, true) ?? [];
        } catch (\Exception $e) {
            Log::error('AppConfig fetch error: ' . $e->getMessage());
            return [];
        }
    }

    private function isInRollout(string $userId, string $flagKey, int $percentage): bool
    {
        $hash = crc32($userId . ':' . $flagKey);
        $bucket = abs($hash) % 100;
        return $bucket < $percentage;
    }

    public function clearCache(): void
    {
        Cache::forget('appconfig_flags');
        $this->flags = [];
    }
}
```

:::message
**AWS SDK認証について**

`CredentialProvider::defaultProvider()` を使用することで、環境変数 `AWS_PROFILE` や `~/.aws/credentials` を自動的にチェックしてくれます。ローカル開発からECS/EC2へのデプロイまで、同じコードで対応できます。
:::

### ルート設定

**src/routes/web.php**:

```php
<?php

use App\Services\FeatureFlagService;
use Illuminate\Support\Facades\Route;

Route::get('/flags', function (FeatureFlagService $service) {
    return response()->json([
        'flags' => $service->getFlags(),
    ]);
});

Route::get('/feature/{flag}', function (FeatureFlagService $service, string $flag) {
    return response()->json([
        'flag' => $flag,
        'enabled' => $service->isEnabled($flag),
    ]);
});

Route::get('/feature/{flag}/user/{userId}', function (FeatureFlagService $service, string $flag, string $userId) {
    return response()->json([
        'flag' => $flag,
        'user_id' => $userId,
        'enabled' => $service->isEnabled($flag, ['user_id' => $userId]),
    ]);
});
```

## Step 4: 基本ON/OFF

### フラグ定義

**appconfig/flags/v1-basic.json**:

```json
{
  "version": "1",
  "flags": {
    "new_dashboard": {
      "name": "New Dashboard",
      "description": "Enable the new dashboard UI"
    },
    "beta_feature": {
      "name": "Beta Feature",
      "description": "Beta feature for testing"
    }
  },
  "values": {
    "new_dashboard": {
      "enabled": true
    },
    "beta_feature": {
      "enabled": false
    }
  }
}
```

### デプロイと確認

```bash
./appconfig/scripts/02-deploy-flags.sh appconfig/flags/v1-basic.json

# キャッシュクリア
docker compose exec app php artisan cache:clear

# 確認
curl -s http://localhost:8000/flags | jq .
```

```json
{
  "flags": {
    "new_dashboard": {
      "enabled": true,
      "allowed_users": []
    },
    "beta_feature": {
      "enabled": false,
      "allowed_users": []
    }
  }
}
```

```bash
curl -s http://localhost:8000/feature/new_dashboard | jq .
# {"flag":"new_dashboard","enabled":true}

curl -s http://localhost:8000/feature/beta_feature | jq .
# {"flag":"beta_feature","enabled":false}
```

## Step 5: ユーザーターゲティング

### フラグ定義

**appconfig/flags/v2-targeting.json**:

```json
{
  "version": "1",
  "flags": {
    "new_dashboard": {
      "name": "New Dashboard",
      "description": "Enable the new dashboard UI"
    },
    "beta_feature": {
      "name": "Beta Feature",
      "description": "Beta feature for specific users only",
      "attributes": {
        "allowed_users": {
          "constraints": {
            "required": false,
            "type": "string"
          }
        }
      }
    }
  },
  "values": {
    "new_dashboard": {
      "enabled": true
    },
    "beta_feature": {
      "enabled": false,
      "allowed_users": "user_123,user_456"
    }
  }
}
```

:::message
**カスタム属性の形式**

Feature Flags型では、カスタム属性は `flags` セクションの `attributes` で定義し、`values` セクションで値を設定します。`allowed_users` はカンマ区切りの文字列として指定します。
:::

### 確認

```bash
./appconfig/scripts/02-deploy-flags.sh appconfig/flags/v2-targeting.json
docker compose exec app php artisan cache:clear

# 対象ユーザー → true
curl -s http://localhost:8000/feature/beta_feature/user/user_123 | jq .
# {"flag":"beta_feature","user_id":"user_123","enabled":true}

# 対象外ユーザー → false
curl -s http://localhost:8000/feature/beta_feature/user/user_789 | jq .
# {"flag":"beta_feature","user_id":"user_789","enabled":false}
```

## Step 6: 段階的ロールアウト

### フラグ定義

**appconfig/flags/v3-rollout.json**:

```json
{
  "version": "1",
  "flags": {
    "new_dashboard": {
      "name": "New Dashboard",
      "description": "Enable the new dashboard UI"
    },
    "beta_feature": {
      "name": "Beta Feature",
      "description": "Beta feature for specific users only",
      "attributes": {
        "allowed_users": {
          "constraints": {
            "required": false,
            "type": "string"
          }
        }
      }
    },
    "gradual_rollout": {
      "name": "Gradual Rollout",
      "description": "Gradual rollout feature",
      "attributes": {
        "rollout_percentage": {
          "constraints": {
            "required": false,
            "type": "number"
          }
        }
      }
    }
  },
  "values": {
    "new_dashboard": {
      "enabled": true
    },
    "beta_feature": {
      "enabled": false,
      "allowed_users": "user_123,user_456"
    },
    "gradual_rollout": {
      "enabled": true,
      "rollout_percentage": 50
    }
  }
}
```

:::message
**段階的ロールアウトについて**

AppConfigのFeature Flags型には、Unleashのような組み込みの段階的ロールアウト機能はありません。`rollout_percentage` をカスタム属性として定義し、アプリケーション側でハッシュベースの判定ロジックを実装しています。
:::

:::message alert
**AppConfigの「デプロイ戦略」との違い**

混同しやすい2つの「段階的ロールアウト」があります：

| 種類 | 説明 | 制御対象 |
|------|------|---------|
| **AppConfigのDeployment Strategy** | 設定ファイルの反映を段階的に行う | サーバー/インスタンス単位 |
| **アプリ側のロールアウト** | 特定ユーザーにのみ新機能を有効化 | ユーザー単位 |

AppConfigのデプロイ戦略（`Linear50PercentEvery30Seconds` など）は「全サーバーのうち最初は10%のサーバーだけに新しい設定値を反映させる」というもので、**個別のユーザー単位の出し分けはアプリ側のロジックが必要**です。
:::

### 確認

```bash
./appconfig/scripts/02-deploy-flags.sh appconfig/flags/v3-rollout.json
docker compose exec app php artisan cache:clear

# 複数ユーザーでテスト（約50%がtrue）
for i in {1..20}; do
  curl -s "http://localhost:8000/feature/gradual_rollout/user/user_$i" | jq -c .
done
```

```
{"flag":"gradual_rollout","user_id":"user_1","enabled":true}
{"flag":"gradual_rollout","user_id":"user_2","enabled":true}
{"flag":"gradual_rollout","user_id":"user_3","enabled":false}
...
```

### Stickiness確認

同じユーザーは何度リクエストしても同じ結果になります。

```bash
curl -s http://localhost:8000/feature/gradual_rollout/user/user_5 | jq .
curl -s http://localhost:8000/feature/gradual_rollout/user/user_5 | jq .
curl -s http://localhost:8000/feature/gradual_rollout/user/user_5 | jq .
# すべて同じ結果
```

:::message
**Stickinessの仕組み**

`user_id` と `flag_key` を組み合わせたハッシュ値（CRC32）を計算し、0〜99の数値に変換してしきい値と比較しています。これはUnleashと同じアプローチで、DBに結果を保存せずステートレスに判定できます。
:::

## Step 7: 環境別管理

### staging環境を作成

```bash
aws appconfig create-environment \
  --application-id <app-id> \
  --name "staging" \
  --description "Staging environment" \
  --region ap-northeast-1
```

### staging用フラグ定義

**appconfig/flags/staging.json**:

```json
{
  "version": "1",
  "flags": {
    "new_dashboard": {
      "name": "New Dashboard",
      "description": "Enable the new dashboard UI"
    },
    "beta_feature": {
      "name": "Beta Feature",
      "description": "Beta feature for specific users only",
      "attributes": {
        "allowed_users": {
          "constraints": {
            "required": false,
            "type": "string"
          }
        }
      }
    },
    "gradual_rollout": {
      "name": "Gradual Rollout",
      "description": "Gradual rollout feature",
      "attributes": {
        "rollout_percentage": {
          "constraints": {
            "required": false,
            "type": "number"
          }
        }
      }
    }
  },
  "values": {
    "new_dashboard": {
      "enabled": false
    },
    "beta_feature": {
      "enabled": false
    },
    "gradual_rollout": {
      "enabled": true,
      "rollout_percentage": 10
    }
  }
}
```

### 環境別デプロイ

```bash
# staging環境にデプロイ
aws appconfig start-deployment \
  --application-id <app-id> \
  --environment-id <staging-env-id> \
  --configuration-profile-id <profile-id> \
  --configuration-version <version> \
  --deployment-strategy-id <strategy-id> \
  --region ap-northeast-1
```

### 環境切り替え

`src/.env` の `APPCONFIG_ENVIRONMENT_ID` を切り替えることで、異なる環境の設定を取得できます。

| フラグ | development | staging |
|--------|-------------|---------|
| `new_dashboard.enabled` | `true` | `false` |
| `gradual_rollout.rollout_percentage` | `50` | `10` |

## ハマりポイント

### 1. Feature Flags型のcontent-type指定（重要）

Feature Flags型をAWS CLIで操作する際、`--content-type` の指定が非常に重要です。

```bash
# ❌ application/json → アップロードは成功するがカスタム属性が無効
--content-type "application/json"

# ✅ Feature Flags型用のcontent-type → カスタム属性も正しく反映
--content-type "application/vnd.amazonaws.appconfig.profiles+json"
```

| content-type | アップロード | Flag value | カスタム属性 |
|--------------|-------------|-----------|-------------|
| `application/json` | ✅ | ❌ 無効 | ❌ 取得不可 |
| `application/vnd.amazonaws.appconfig.profiles+json` | ✅ | ✅ 有効 | ✅ 取得可能 |

正しいcontent-typeを指定すれば、**AWS CLIのみでFeature Flags型のフラグ管理が完結**できます。

### 2. デプロイのBake time

デフォルトの `AppConfig.AllAtOnce` でも `FinalBakeTimeInMinutes: 10` が設定されています。検証時は即時デプロイ用のカスタム戦略を作成することを推奨します。

### 3. コンテナ内でのAWS認証

AWS SDKは以下の順序で認証情報を探します：

1. 環境変数（`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`）
2. `AWS_PROFILE` + `~/.aws/credentials` ファイル
3. IAM Role（EC2/ECS上）

`CredentialProvider::defaultProvider()` を使えば、この順序で自動的に認証情報を探してくれます。コンテナに `~/.aws` をマウントし、`AWS_PROFILE` 環境変数を設定すれば動作します。

### 4. デプロイ中は新しいデプロイを開始できない

`DEPLOYING` または `BAKING` 状態のデプロイがあると、新しいデプロイは開始できません。

```bash
# 現在のデプロイを停止
aws appconfig stop-deployment \
  --application-id <app-id> \
  --environment-id <env-id> \
  --deployment-number <number> \
  --region ap-northeast-1
```

## ベストプラクティス

### NextPollConfigurationTokenの活用

本記事の実装では簡易的に毎回 `startConfigurationSession` を呼んでいますが、本番環境では以下のベストプラクティスが推奨されます。

**推奨パターン：**

1. **セッションの維持**: 同一セッション内で `NextPollConfigurationToken` を回し続ける
2. **差分取得**: 設定に変更がない場合、`getLatestConfiguration` は空のコンテンツを返す（データ転送量削減）

```php
// 本番向け実装例（バックグラウンドポーリング）
class FeatureFlagPoller
{
    private ?string $nextToken = null;
    private array $cachedFlags = [];

    public function poll(): void
    {
        if ($this->nextToken === null) {
            // 初回：セッション開始
            $session = $this->client->startConfigurationSession([...]);
            $this->nextToken = $session['InitialConfigurationToken'];
        }

        // 設定を取得
        $response = $this->client->getLatestConfiguration([
            'ConfigurationToken' => $this->nextToken,
        ]);

        // 次回用トークンを保存
        $this->nextToken = $response['NextPollConfigurationToken'];

        // コンテンツがある場合のみ更新（変更がない場合は空）
        $content = $response['Configuration']->getContents();
        if (!empty($content)) {
            $this->cachedFlags = json_decode($content, true);
        }
    }
}
```

:::message
**本記事の実装について**

本記事では検証・学習目的のため、シンプルな実装（毎回セッション開始 + Laravelキャッシュ）を採用しています。本番環境では上記のポーリングパターンや、AWS公式の[AppConfig Agent](https://docs.aws.amazon.com/appconfig/latest/userguide/appconfig-integration-lambda-extensions.html)の利用を検討してください。
:::

## 本番運用時のIAMポリシー

ECSやEC2で運用する際に必要な最小権限のIAMポリシー例です。

### アプリケーション用（設定取得のみ）

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "appconfig:StartConfigurationSession",
        "appconfig:GetLatestConfiguration"
      ],
      "Resource": [
        "arn:aws:appconfig:ap-northeast-1:123456789012:application/*/environment/*/configuration/*"
      ]
    }
  ]
}
```

### 管理者用（設定の更新・デプロイ）

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "appconfig:CreateHostedConfigurationVersion",
        "appconfig:StartDeployment",
        "appconfig:StopDeployment",
        "appconfig:GetDeployment",
        "appconfig:ListDeployments"
      ],
      "Resource": [
        "arn:aws:appconfig:ap-northeast-1:123456789012:application/*"
      ]
    }
  ]
}
```

:::message
**リソースの絞り込み**

本番環境では、`*` の部分を実際のApplication ID、Environment ID、Configuration Profile IDに置き換えて、アクセス範囲を限定することを推奨します。
:::

## 検証結果まとめ

| 観点 | 評価 | コメント |
|------|------|----------|
| **導入の容易さ** | ○ | AWS CLI + SDKで構築可能。ただしリソース理解に時間がかかる |
| **コードの書き心地** | ○ | セッション管理が必要だが、SDK自体はシンプル。ターゲティング・ロールアウトはアプリ側実装 |
| **管理UI** | ○ | AWSコンソールで操作可能。Feature Flags型ならUIで直感的に管理できる |
| **ユーザーターゲティング** | ○ | Feature Flags型のカスタム属性 + アプリ側ロジックで対応可能 |
| **段階的ロールアウト** | △ | アプリ側でハッシュ計算を実装 |
| **環境別管理** | ◎ | Environment単位で明確に分離。IAMで権限制御も可能 |
| **運用負荷** | ○ | デプロイ戦略の設定が必要。ロールバック機能あり |
| **追加インフラ** | ◎ | AWSマネージド。追加構築不要 |
| **コスト** | ○ | 設定取得回数に応じた課金。キャッシュ戦略が重要 |

## 3ツール比較

| 観点 | Pennant | Unleash | AppConfig |
|------|---------|---------|-----------|
| **導入の容易さ** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **管理UI** | ❌ なし | ✅ 標準搭載 | ○ AWSコンソール |
| **非エンジニア操作** | ❌ 難しい | ✅ 可能 | △ AWS知識が必要 |
| **ターゲティング** | ✅ 組み込み | ✅ 組み込み | ○ カスタム属性 + アプリ実装 |
| **ロールアウト** | △ 自前実装 | ✅ 組み込み | △ 自前実装 |
| **環境別管理** | △ .env切り替え | ✅ UI上で管理 | ✅ Environment分離 |
| **追加インフラ** | なし | PostgreSQL + サーバー | なし（AWS） |
| **Stickiness** | DB保存 | ハッシュ計算 | ハッシュ計算 |
| **反映速度** | 即時 | 数秒〜数十秒 | キャッシュ依存 |
| **コスト** | 無料 | OSS無料 / Cloud有料 | 従量課金 |
| **スケーラビリティ** | △ DBに依存 | ◎ ステートレス | ◎ AWSマネージド |

## 所感

### AppConfigが向いているケース

- **すでにAWSを利用している**プロジェクトで、追加インフラを増やしたくない
- **環境別管理とIAM権限制御**を厳密に行いたい
- **段階的デプロイとロールバック**機能を活用したい
- エンジニアがJSON編集でフラグ管理することに抵抗がない

### AppConfigが向いていないケース

- 非エンジニアがGUIでフラグを直感的に操作したい → **Unleash**
- シンプルにLaravelだけで完結させたい → **Pennant**
- 複雑なターゲティングルールをUI上で設定したい → **Unleash**

### 総評

AppConfigは「AWSエコシステム内でFeature Flagを管理したい」というニーズには応えられますが、**Feature Flag専用ツールとしては機能が限定的**です。

特にターゲティングや段階的ロールアウトは自前実装が必要で、Unleashのような専用ツールと比べると開発工数がかかります。一方、AWSマネージドである点、IAMによる権限管理、既存のAWSインフラとの親和性は大きなメリットです。

**結論**: AWS環境で「シンプルなON/OFF + 環境別管理」がメインなら十分実用的。高度なターゲティングやA/Bテストが必要ならUnleashを検討。

## 参考

- [AWS AppConfig 公式ドキュメント](https://docs.aws.amazon.com/appconfig/)
- [AWS SDK for PHP - AppConfigData](https://docs.aws.amazon.com/aws-sdk-php/v3/api/class-Aws.AppConfigData.AppConfigDataClient.html)
- [Feature flags in AWS AppConfig](https://docs.aws.amazon.com/appconfig/latest/userguide/appconfig-creating-configuration-and-profile-feature-flags.html)
