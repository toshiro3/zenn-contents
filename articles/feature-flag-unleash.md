---
title: "Unleash OSSでFeature Flagを試してみた【Laravel連携】"
emoji: "🚀"
type: "tech"
topics: ["laravel", "php", "unleash", "featureflag", "docker"]
published: true
---

## はじめに

Feature Flagツール比較検証の第2弾として、**Unleash OSS**（セルフホスト版）を検証しました。

前回のLaravel Pennant検証はこちら：
https://zenn.dev/toshiro3/articles/feature-flag-laravel-pennant

### Unleashとは

Unleash は、オープンソースの Feature Flag 管理プラットフォームです。

- **OSS版**（セルフホスト）と**クラウド版**（マネージド）がある
- **管理UI**が標準搭載されており、非エンジニアでも操作可能
- 多言語SDK対応（PHP, Node.js, Python, Go, etc.）
- 環境別管理、段階的ロールアウト、ユーザーターゲティングなどの機能が組み込み

## 検証環境

| 項目 | バージョン |
|------|-----------|
| Laravel | 12.x |
| PHP | 8.3 |
| Unleash | latest（v5系） |
| laravel-unleash | 2.9.0 |
| Docker | 27.x |

### リポジトリ

検証用のソースコードはこちら：
https://github.com/toshiro3/laravel-unleash-demo

## Step 1: 環境構築（Docker Compose）

Unleash OSS + Laravel を Docker Compose で構築します。

### ディレクトリ構成

```
laravel-unleash-demo/
├── docker-compose.yml
├── docker/
│   └── frankenphp/
│       ├── Dockerfile
│       ├── Caddyfile
│       └── entrypoint.sh
└── src/                    # Laravelアプリケーション
```

### docker-compose.yml

```yaml
services:
  # Laravel Application (FrankenPHP)
  app:
    build:
      context: .
      dockerfile: docker/frankenphp/Dockerfile
    container_name: laravel-app
    ports:
      - "8000:8000"
    volumes:
      - ./src:/app
    depends_on:
      mysql:
        condition: service_healthy
      unleash:
        condition: service_healthy
    environment:
      SERVER_NAME: ":8000"
    networks:
      - app-network

  # MySQL (Laravel用)
  mysql:
    image: mysql:8.0
    container_name: laravel-mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: laravel
      MYSQL_USER: laravel
      MYSQL_PASSWORD: laravel
    volumes:
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  # PostgreSQL (Unleash用)
  postgres:
    image: postgres:15
    container_name: unleash-postgres
    environment:
      POSTGRES_USER: unleash
      POSTGRES_PASSWORD: unleash
      POSTGRES_DB: unleash
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U unleash"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  # Unleash Server
  unleash:
    image: unleashorg/unleash-server:latest
    container_name: unleash-server
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://unleash:unleash@postgres:5432/unleash
      DATABASE_SSL: "false"
      AUTH_TYPE: open-source
      LOG_LEVEL: info
      INIT_ADMIN_API_TOKENS: "*:*.unleash-demo-api-token"
      INIT_CLIENT_API_TOKENS: "default:development.unleash-demo-client-token,default:production.unleash-prod-client-token"
    ports:
      - "4242:4242"
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:4242/health"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

volumes:
  mysql_data:
  postgres_data:

networks:
  app-network:
    driver: bridge
```

### .env（Laravel）

アプリケーションの設定は `src/.env` で管理します：

```dotenv
APP_NAME=Laravel
APP_ENV=local
APP_DEBUG=true

# Database
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=laravel
DB_PASSWORD=laravel

# Unleash
UNLEASH_URL=http://unleash:4242/api
UNLEASH_APP_NAME=laravel-unleash-demo
UNLEASH_INSTANCE_ID=local-dev
UNLEASH_API_KEY=default:development.unleash-demo-client-token
UNLEASH_ENVIRONMENT=development
```

### 認証モードについて

`AUTH_TYPE` の設定によって認証方式が変わります。

| AUTH_TYPE | 用途 | ユーザー管理 | APIトークン作成 |
|-----------|------|-------------|----------------|
| `demo` | 検証・デモ | ❌ | ❌（初期トークンのみ） |
| `open-source` | 本番運用（OSS） | ✅ | ✅ |
| `enterprise` | 本番運用（有料） | ✅ + SSO/SAML | ✅ |

本番運用では `open-source` を使用し、初回ログイン後にadminパスワードを変更することを推奨します。

- デフォルトadmin: `admin` / `unleash4all`

### 起動

```bash
docker compose up -d --build
```

- Laravel: http://localhost:8000
- Unleash UI: http://localhost:4242

## Step 2: Laravel連携

### パッケージインストール

```bash
docker run --rm -v $(pwd)/src:/app composer require j-webb/laravel-unleash -W --ignore-platform-reqs
```

:::message
Laravel 12 と `laravel-unleash` の依存関係の問題で `-W` オプションが必要になる場合があります。
:::

### 設定ファイル

`src/config/unleash.php` を作成：

```php
<?php

return [
    'enabled' => env('UNLEASH_ENABLED', true),
    'url' => env('UNLEASH_URL'),
    'instance_id' => env('UNLEASH_INSTANCE_ID', 'default'),
    'environment' => env('UNLEASH_ENVIRONMENT', 'development'),
    'automatic_registration' => env('UNLEASH_AUTOMATIC_REGISTRATION', false),
    'metrics' => env('UNLEASH_METRICS', false),
    
    'cache' => [
        'enabled' => env('UNLEASH_CACHE_ENABLED', false),
        'ttl' => env('UNLEASH_CACHE_TTL', 30),
        'handler' => \JWebb\Unleash\Cache\CacheHandler::class,
    ],

    'http_client_override' => [
        'enabled' => env('UNLEASH_HTTP_CLIENT_OVERRIDE', false),
        'config' => [
            'timeout' => env('UNLEASH_HTTP_TIMEOUT', 5),
            'connect_timeout' => env('UNLEASH_HTTP_CONNECT_TIMEOUT', 5),
        ],
    ],

    'api_key' => env('UNLEASH_API_KEY'),
    
    'strategy_provider' => \JWebb\Unleash\Providers\UnleashStrategiesProvider::class,
    'context_provider' => \JWebb\Unleash\Providers\UnleashContextProvider::class,
];
```

:::message alert
`url` には `/api` を含める必要があります（例: `http://unleash:4242/api`）
:::

:::message
**SDKのキャッシュについて**

Unleash SDKはパフォーマンス向上のため、フラグ定義をバックグラウンドで定期取得しメモリに保持します。そのため、UIで変更してから反映されるまでデフォルトで数秒〜数十秒のラグが生じます。

検証中に即時反映を確認したい場合は、`UNLEASH_CACHE_ENABLED=false`（上記設定のデフォルト）にしてください。本番環境ではキャッシュを有効化し、TTLを適切に設定することを推奨します。
:::

### ルート設定

`src/routes/web.php`:

```php
<?php

use Illuminate\Support\Facades\Route;
use JWebb\Unleash\Facades\Unleash;
use Unleash\Client\Configuration\UnleashContext;

Route::get('/', function () {
    return view('welcome');
});

// Unleash接続確認
Route::get('/unleash-test', function () {
    return response()->json([
        'message' => 'Unleash connection test',
        'unleash_url' => config('unleash.url'),
    ]);
});

// フラグ状態取得
Route::get('/flags/{flag}', function (string $flag) {
    $userId = request()->query('user_id');

    $context = null;
    if ($userId) {
        $context = (new UnleashContext())
            ->setCurrentUserId($userId);
    }

    $isEnabled = Unleash::isEnabled($flag, $context);

    return response()->json([
        'flag' => $flag,
        'user_id' => $userId,
        'is_active' => $isEnabled,
    ]);
});
```

### 接続確認

```bash
curl -s http://localhost:8000/unleash-test | jq .
```

```json
{
  "message": "Unleash connection test",
  "unleash_url": "http://unleash:4242/api"
}
```

## Step 3: 基本ON/OFF

### Unleash UIでフラグを作成

1. http://localhost:4242 にアクセス
2. 「Projects」→「Default」→「New feature flag」
3. Name: `new-feature`、Description: `テスト用フラグ`
4. 「Create feature flag」

### ストラテジーを設定

フラグ作成直後は無効です。有効化するには：

1. **development** 環境で「Add strategy」
2. 「Standard」を選択
3. 「Save strategy」
4. **development環境のトグルをON**

### 確認

```bash
curl -s http://localhost:8000/flags/new-feature | jq .
```

```json
{
  "flag": "new-feature",
  "user_id": null,
  "is_active": true
}
```

:::message
フラグ変更はSDKのポーリング間隔（数秒〜数十秒）で反映されます。これはUnleashの設計思想で、毎リクエストでサーバーに問い合わせるのではなく、バックグラウンドでフラグ定義を取得・保持することでパフォーマンスを向上させています。
:::

## Step 4: ユーザーターゲティング

特定のユーザーIDのみフラグを有効化します。

### Unleash UIでConstraintを設定

1. `new-feature` フラグの「Standard」ストラテジーを編集
2. 「Targeting」タブ →「Add constraint」
3. 設定：
   - **Context field**: `userId`
   - **Operator**: `is one of`
   - **Values**: `123`, `456`
4. 「Save strategy」

:::message
**userIdは文字列として扱われます**

Unleashの`userId`は内部的に文字列（String）として処理されます。UIで設定する値も文字列として扱われるため、コード側でも `(string) $user->id` のように明示的にキャストすることを推奨します。
:::

### 確認

```bash
# 対象ユーザー → true
curl -s "http://localhost:8000/flags/new-feature?user_id=123" | jq .
# {"flag":"new-feature","user_id":"123","is_active":true}

# 対象外ユーザー → false
curl -s "http://localhost:8000/flags/new-feature?user_id=999" | jq .
# {"flag":"new-feature","user_id":"999","is_active":false}

# user_idなし → false
curl -s "http://localhost:8000/flags/new-feature" | jq .
# {"flag":"new-feature","user_id":null,"is_active":false}
```

### 実運用でのContext設定

検証ではクエリストリングでuser_idを渡しましたが、実運用では認証情報から取得します：

```php
use Illuminate\Support\Facades\Auth;
use JWebb\Unleash\Facades\Unleash;
use Unleash\Client\Configuration\UnleashContext;

$user = Auth::user();

$context = null;
if ($user) {
    $context = (new UnleashContext())
        ->setCurrentUserId((string) $user->id)
        ->setCustomProperty('plan', $user->plan)  // カスタム属性も可能
        ->setCustomProperty('company_id', $user->company_id);
}

$isEnabled = Unleash::isEnabled('premium-feature', $context);
```

## Step 5: 段階的ロールアウト

パーセンテージ制御と、同一ユーザーへの一貫性（Stickiness）を確認します。

### 新しいフラグを作成

1. 「New feature flag」→ Name: `gradual-rollout`
2. 「Add strategy」→「Gradual rollout」
3. 設定：
   - **Rollout**: 50%
   - **Stickiness**: Default（userId優先）
4. 「Save strategy」
5. **development環境のトグルをON**

### 確認

```bash
# 複数ユーザーで試す（約半数がtrueになる）
for i in {1..20}; do
  curl -s "http://localhost:8000/flags/gradual-rollout?user_id=$i" | jq -c .
done
```

```
{"flag":"gradual-rollout","user_id":"1","is_active":false}
{"flag":"gradual-rollout","user_id":"2","is_active":false}
{"flag":"gradual-rollout","user_id":"3","is_active":true}
{"flag":"gradual-rollout","user_id":"4","is_active":true}
...
```

### Stickiness確認

同じユーザーは常に同じ結果が返ります：

```bash
curl -s "http://localhost:8000/flags/gradual-rollout?user_id=5" | jq .
curl -s "http://localhost:8000/flags/gradual-rollout?user_id=5" | jq .
curl -s "http://localhost:8000/flags/gradual-rollout?user_id=5" | jq .
# すべて同じ結果（true or false）
```

### Stickinessの仕組み

Unleashの段階的ロールアウトは、`userId` と `flagName` を組み合わせたハッシュ値を計算し、0〜100の数値に変換してしきい値と比較しています。

```
hash(userId + flagName) % 100 < rollout% → true
```

**この仕組みの利点**：
- サーバー側に状態を保存する必要がない（ステートレス）
- 同じユーザーに対して常に同じ結果を返せる
- DBへの書き込みが発生しないためスケーラブル

これは**Pennant（DBに結果を保存）との大きな違い**です。Pennantはユーザーごとの結果をDBに保存するため、ロールアウト率を変更しても既存ユーザーの結果は変わりません。一方、Unleashはハッシュ計算のため、ロールアウト率を変更すると対象ユーザーが変わる可能性があります。

## Step 6: 環境別管理

development / production で異なる状態を管理できるか確認します。

### 環境ごとのAPIトークン

Unleashでは、**環境ごとに異なるAPIトークン**を使用します。

```yaml
# docker-compose.yml の INIT_CLIENT_API_TOKENS
INIT_CLIENT_API_TOKENS: "default:development.unleash-demo-client-token,default:production.unleash-prod-client-token"
```

### Laravel側の設定（.env）

```dotenv
# development環境
UNLEASH_ENVIRONMENT=development
UNLEASH_API_KEY=default:development.unleash-demo-client-token

# production環境（本番デプロイ時）
UNLEASH_ENVIRONMENT=production
UNLEASH_API_KEY=default:production.unleash-prod-client-token
```

### 確認

UIで `new-feature` を development=ON、production=OFF に設定した場合：

```bash
# development環境 → true
curl -s "http://localhost:8000/flags/new-feature?user_id=123" | jq .

# production環境（設定変更後）→ false
curl -s "http://localhost:8000/flags/new-feature?user_id=123" | jq .
```

## ハマりポイント

### 1. URLに `/api` が必要

```php
// ❌ 403エラーになる
'url' => 'http://unleash:4242'

// ✅ 正しい
'url' => 'http://unleash:4242/api'
```

### 2. 設定キーは `api_key`（`api_token` ではない）

```php
// ❌ パッケージのデフォルト設定と異なる
'api_token' => env('UNLEASH_API_TOKEN'),

// ✅ 正しい（laravel-unleash 2.9.0）
'api_key' => env('UNLEASH_API_KEY'),
```

### 3. Contextは配列ではなくオブジェクト

```php
// ❌ TypeError
Unleash::isEnabled('flag', ['userId' => '123']);

// ✅ 正しい
$context = (new UnleashContext())->setCurrentUserId('123');
Unleash::isEnabled('flag', $context);
```

### 4. Strategyを設定しないとtrueにならない

フラグを作成しただけではONにならず、環境ごとに「Strategy」を追加してトグルをONにする必要があります。

## まとめ

### 検証結果

| シナリオ | 結果 |
|----------|------|
| 基本ON/OFF | ✅ UIでトグル操作、コードから取得可能 |
| ユーザーターゲティング | ✅ Constraintで柔軟に設定可能 |
| 段階的ロールアウト | ✅ パーセンテージ制御、Stickiness対応 |
| 環境別管理 | ✅ 環境ごとにトークンとストラテジーを分離 |

### Pennantとの比較

| 観点 | Pennant | Unleash |
|------|---------|---------|
| **導入の容易さ** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **管理UI** | ❌ なし | ✅ 標準搭載 |
| **非エンジニア操作** | ❌ 難しい | ✅ 可能 |
| **フラグ変更** | コード変更 + デプロイ | UIで即時変更 |
| **段階的ロールアウト** | 自前実装 | 組み込み機能 |
| **Stickinessの仕組み** | DBに保存（ステートフル） | ハッシュ計算（ステートレス） |
| **環境別管理** | .env切り替え | UI上で視覚的に管理 |
| **追加インフラ** | なし | PostgreSQL + Unleashサーバー |
| **反映速度** | 即時 | 数秒〜数十秒 |
| **コスト** | 無料 | OSS無料 / Cloud有料 |

### 所感

**Unleashが向いているケース**：
- 非エンジニア（PM、ディレクター）がフラグを操作する必要がある
- 段階的ロールアウトやA/Bテストを頻繁に行う
- 複数環境でのフラグ状態を視覚的に管理したい
- フラグ変更のたびにデプロイしたくない

**Pennantが向いているケース**：
- エンジニアのみがフラグを操作する
- シンプルなON/OFFで十分
- 追加インフラを増やしたくない
- 即時反映が必須

### 注意点

1. **URLに `/api` を含める**: `laravel-unleash` パッケージでは `UNLEASH_URL` に `/api` パスを含める必要がある
2. **認証モード**: 本番運用では `AUTH_TYPE: open-source` を使用し、ユーザー管理を有効化する
3. **ポーリング間隔**: フラグ変更は即時反映ではなく、数秒〜数十秒のラグがある
4. **依存関係**: Laravel 12 では `composer require` 時に `-W` オプションが必要になる場合がある

## 次回

次回は **AWS AppConfig** を検証予定です。
