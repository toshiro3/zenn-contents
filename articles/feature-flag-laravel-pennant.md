---
title: "Laravel Pennantで機能フラグを試してみた"
emoji: "🚩"
type: "tech"
topics: ["laravel", "php", "featureflag"]
published: true
---

## Laravel Pennantとは

Laravel PennantはLaravel公式の機能フラグ（Feature Flag）パッケージです。機能フラグを使うと、コードをデプロイした状態で特定の機能のON/OFFを制御できます。

**主な用途：**
- 新機能の段階的リリース
- 特定ユーザーのみへの機能開放（ベータテスト）
- A/Bテスト
- トランクベース開発でのフラグ管理

## 検証環境

Docker Composeを使って検証環境を構築しました。

| 項目 | バージョン |
|------|-----------|
| PHP | 8.4 |
| Laravel | 12.x |
| MySQL | 8.0 |
| Laravel Pennant | 1.x |

検証用コードはGitHubで公開しています。
https://github.com/toshiro3/laravel-pennant-demo

## 環境構築

### プロジェクト構成

```
laravel-pennant-demo/
├── docker-compose.yml
├── docker/
│   └── php/
│       ├── Dockerfile
│       └── php.ini
└── src/
    └── (Laravelプロジェクト)
```

### Dockerfile

```dockerfile
FROM php:8.4-cli

RUN apt-get update && apt-get install -y \
    git \
    unzip \
    libzip-dev \
    && docker-php-ext-install pdo pdo_mysql zip

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

EXPOSE 8000

CMD ["php", "artisan", "serve", "--host=0.0.0.0"]
```

### docker-compose.yml

```yaml
services:
  app:
    build:
      context: .
      dockerfile: docker/php/Dockerfile
    ports:
      - "8000:8000"
    volumes:
      - ./src:/var/www/html
      - ./docker/php/php.ini:/usr/local/etc/php/conf.d/local.ini
    depends_on:
      db:
        condition: service_healthy

  db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: pennant_demo
      MYSQL_USER: laravel
      MYSQL_PASSWORD: secret
      MYSQL_ROOT_PASSWORD: rootsecret
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 5s
      retries: 10

volumes:
  db_data:
```

### .envの修正

Laravel 12ではデフォルトのDBがSQLiteになっています。MySQLを使用するため、`src/.env`のDB設定を修正します。

```
DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=pennant_demo
DB_USERNAME=laravel
DB_PASSWORD=secret
```

### Pennantのインストール

```bash
# Laravelプロジェクト作成
docker run --rm -v $(pwd)/src:/app composer create-project laravel/laravel /app

# コンテナ起動
docker compose up -d --build

# Pennantインストール
docker compose exec app composer require laravel/pennant

# 設定ファイル・マイグレーションをパブリッシュ
docker compose exec app php artisan vendor:publish --provider="Laravel\Pennant\PennantServiceProvider"

# マイグレーション実行
docker compose exec app php artisan migrate
```

これだけで導入完了です。`features`テーブルが作成され、フラグの状態が永続化されます。

## 基本的な使い方

### Featureクラスの作成

```bash
docker compose exec app php artisan make:class Features/NewDashboard
```

```php
<?php

namespace App\Features;

class NewDashboard
{
    public function resolve(): bool
    {
        return false; // デフォルトはOFF
    }
}
```

`resolve()`メソッドがフラグの初期値を決定します。

### APIでフラグの状態を確認

```php
use Laravel\Pennant\Feature;
use App\Features\NewDashboard;

Route::get('/feature/new-dashboard', function () {
    return response()->json([
        'feature' => 'new-dashboard',
        'active' => Feature::active(NewDashboard::class),
    ]);
});
```

```bash
$ curl -s http://localhost:8000/api/feature/new-dashboard | jq .
{
  "feature": "new-dashboard",
  "active": false
}
```

### フラグのON/OFF切り替え

```php
// 全員ON
Route::post('/feature/new-dashboard/activate', function () {
    Feature::activateForEveryone(NewDashboard::class);
    return response()->json(['status' => 'activated for everyone']);
});

// 全員OFF
Route::post('/feature/new-dashboard/deactivate', function () {
    Feature::deactivateForEveryone(NewDashboard::class);
    return response()->json(['status' => 'deactivated for everyone']);
});
```

```bash
# ONにする
$ curl -s -X POST http://localhost:8000/api/feature/new-dashboard/activate | jq .
{
  "status": "activated for everyone"
}

# 状態確認
$ curl -s http://localhost:8000/api/feature/new-dashboard | jq .
{
  "feature": "new-dashboard",
  "active": true
}
```

管理UIは用意されていないため、APIやコマンド経由で操作します。

## ユーザーターゲティング

特定のユーザーにのみフラグを有効化できます。

### 実装

```php
use App\Models\User;

// 特定ユーザーのみON
Route::post('/feature/new-dashboard/activate-user/{userId}', function (int $userId) {
    $user = User::findOrFail($userId);
    Feature::for($user)->activate(NewDashboard::class);
    return response()->json(['status' => 'activated', 'user_id' => $userId]);
});

// ユーザー指定でフラグ確認
Route::get('/feature/new-dashboard', function (Request $request) {
    $scope = $request->query('user_id')
        ? User::find($request->query('user_id'))
        : null;

    return response()->json([
        'feature' => 'new-dashboard',
        'active' => Feature::for($scope)->active(NewDashboard::class),
        'scope' => $scope ? 'user:' . $scope->id : 'global',
    ]);
});
```

### 動作確認

```bash
# user_id=1 のみON
$ curl -s -X POST http://localhost:8000/api/feature/new-dashboard/activate-user/1 | jq .
{
  "status": "activated",
  "user_id": 1
}

# user_id=1 はON
$ curl -s "http://localhost:8000/api/feature/new-dashboard?user_id=1" | jq .
{
  "feature": "new-dashboard",
  "active": true,
  "scope": "user:1"
}

# user_id=2 はOFF
$ curl -s "http://localhost:8000/api/feature/new-dashboard?user_id=2" | jq .
{
  "feature": "new-dashboard",
  "active": false,
  "scope": "user:2"
}
```

### DBの状態

`features`テーブルでは、`scope`カラムでユーザーごとに状態を管理しています。

```
+----+---------------------------+-------------------+-------+
| id | name                      | scope             | value |
+----+---------------------------+-------------------+-------+
|  1 | App\Features\NewDashboard | __laravel_null    | false |
|  2 | App\Features\NewDashboard | App\Models\User|1 | true  |
|  3 | App\Features\NewDashboard | App\Models\User|2 | false |
+----+---------------------------+-------------------+-------+
```

ユーザー数 × フラグ数でレコードが増えるため、アクセス頻度の高い機能ではDBへの負荷に注意が必要です。

## 段階的ロールアウト

ユーザーの一定割合にのみフラグをONにする機能です。

### Featureクラス

```php
<?php

namespace App\Features;

use Illuminate\Support\Lottery;
use App\Models\User;

class GradualRollout
{
    public function resolve(User $user): bool
    {
        $percentage = config('features.gradual_rollout_percentage', 30);
        return Lottery::odds($percentage, 100)->choose();
    }
}
```

`Lottery::odds(30, 100)`で30%の確率でtrueを返します。

### Stickinessの確認

同じユーザーは常に同じ結果になります。

```bash
# user_id=1 は何度叩いても同じ結果
$ curl -s "http://localhost:8000/api/feature/gradual-rollout?user_id=1" | jq .active
false
$ curl -s "http://localhost:8000/api/feature/gradual-rollout?user_id=1" | jq .active
false
$ curl -s "http://localhost:8000/api/feature/gradual-rollout?user_id=1" | jq .active
false

# user_id=3 は別の結果（今回はtrue）
$ curl -s "http://localhost:8000/api/feature/gradual-rollout?user_id=3" | jq .active
true
```

初回アクセス時に`Lottery`が評価され、結果がDBに保存されます。2回目以降はDBの値を参照するため、同じ結果が返ります。

### ロールアウト率を変更する場合

既存レコードを削除してから、`resolve()`の確率を変更する必要があります。

```bash
docker compose exec app php artisan tinker --execute="DB::table('features')->where('name', 'App\\\Features\\\GradualRollout')->delete();"
```

この点は運用上やや手間です。

## 環境別管理

環境（dev/staging/production）によってフラグのデフォルト値を変えたい場合は、config経由で環境変数を読み取る方式が実用的です。

### config/features.php

```php
<?php

return [
    'new_dashboard' => env('FEATURE_NEW_DASHBOARD', false),
    'gradual_rollout_percentage' => (int) env('FEATURE_GRADUAL_ROLLOUT_PERCENTAGE', 30),
];
```

### Featureクラス

```php
<?php

namespace App\Features;

class NewDashboard
{
    public function resolve(): bool
    {
        return config('features.new_dashboard', false);
    }
}
```

### .env

```
# dev環境
FEATURE_NEW_DASHBOARD=true

# production環境
FEATURE_NEW_DASHBOARD=false
```

環境ごとに`.env`を分けることで、デプロイ先に応じたフラグ管理ができます。

**注意点：** DBに保存された値が優先されるため、環境変数を変更しても既存のフラグ状態は変わりません。環境変数の変更を反映するには、該当するフラグのレコードを削除する必要があります。

## 検証結果まとめ

| 観点 | 評価 | コメント |
|------|------|----------|
| **導入の容易さ** | ◎ | `composer require` + `migrate`で即利用可能 |
| **コードの書き心地** | ◎ | `Feature::active()`, `Feature::for($user)` など直感的なAPI |
| **管理UI** | △ | なし。API/コマンド/tinkerで操作 |
| **ユーザーターゲティング** | ◎ | `Feature::for($user)` で簡単に実装 |
| **段階的ロールアウト** | ○ | `Lottery`で実装可能。ただし確率変更時は既存データ削除が必要 |
| **環境別管理** | ○ | config + 環境変数で対応可能 |
| **運用負荷** | △ | 管理UIがないため、運用ツールを自前で用意する必要あり |
| **パフォーマンス** | △ | ユーザー単位でDBアクセスが発生。大規模利用時はRedis等のインメモリキャッシュの検討が必要 |

### 向いているケース

- Laravelプロジェクトでシンプルに機能フラグを導入したい
- 開発チーム内でのフラグ管理が主目的（非エンジニアの操作が不要）
- 小〜中規模のユーザー数

### 向いていないケース

- 非エンジニアがGUIでフラグを操作したい
- 大規模ユーザーへのパーセンテージロールアウトを頻繁に行う
- 複数サービス間でフラグを共有したい

次回は、Unleash（OSS）を検証してみようと思います。

## 参考

- [Laravel Pennant 公式ドキュメント](https://laravel.com/docs/pennant)
- [GitHub: laravel/pennant](https://github.com/laravel/pennant)
