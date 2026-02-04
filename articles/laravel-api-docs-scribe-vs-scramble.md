---
title: "LaravelのAPIドキュメント生成ツール比較：Scribe vs Scramble"
emoji: "📄"
type: "tech"
topics: ["Laravel", "PHP", "API", "OpenAPI"]
published: false
---

## はじめに

LaravelでREST APIを開発する際、APIドキュメントの自動生成ツールとして「Scribe」と「Scramble」が代表的です。本記事では、両ツールを検証し、それぞれの特徴と違いについて比較します。

### 検証環境

| 項目 | バージョン |
|------|-----------|
| PHP | 8.4 |
| Laravel | 12 |
| Scribe | 5.x |
| Scramble | 0.13.x |

検証に使用したコードは以下のリポジトリで公開しています。

https://github.com/toshiro3/laravel-api-docs-comparison

---

## 各ツールの概要

### Scribe

Scribeは、PHPDocアノテーションベースでAPIドキュメントを生成するツールです。

**主な特徴:**
- `@response`, `@responseField`, `@queryParam` などのアノテーションで詳細に記述
- `response_calls` 機能で実際のAPIを呼び出してレスポンス例を取得
- Postmanコレクションの自動生成
- コマンド実行（`php artisan scribe:generate`）でドキュメント生成

### Scramble

Scrambleは、PHPDoc不要でOpenAPIドキュメントを自動生成するツールです。

**主な特徴:**
- コードからの自動推論（FormRequest、API Resourceの型情報を解析）
- リアルタイム生成（コマンド実行不要、アクセス時に動的生成）
- 静的解析ベース（DBアクセス不要）
- OpenAPI 3.1.0対応

### インストール

**Scribe:**
```bash
composer require knuckleswtf/scribe
php artisan vendor:publish --tag=scribe-config
```

**Scramble:**
```bash
composer require dedoc/scramble
# 設定ファイルの公開（オプション）
php artisan vendor:publish --provider="Dedoc\Scramble\ScrambleServiceProvider" --tag="scramble-config"
```

Scrambleはインストール後、`/docs/api` にアクセスするだけでドキュメントが表示されます。

---

## Scribeの response_calls について

Scribeには `response_calls` という機能があり、ドキュメント生成時に実際のAPIを呼び出してレスポンス例を取得できます。

### response_calls の動作

```php
// config/scribe.php
'response_calls' => [
    'methods' => ['GET', 'POST', 'PUT', 'DELETE'],
],
```

この設定の場合、`php artisan scribe:generate` 実行時に**実際にHTTPリクエストを送信**します。

| メソッド | 実行される処理 |
|---------|---------------|
| GET /posts | 実際にDB検索 |
| POST /posts | 実際にバリデーション実行 |
| GET /posts/{post} | 実際にDBから取得 |

:::message alert
`response_calls` は実際にリクエストを送信するため、**副作用のある処理**（メール送信、外部API連携、課金処理など）が含まれるエンドポイントには注意が必要です。本番環境での実行は避け、テスト用の環境で使用することを推奨します。
:::

### response_calls を使わない方法

`response_calls` を使わずにレスポンス例を定義する場合は、`@response` アノテーションを使用します。

```php
// config/scribe.php
'response_calls' => [
    'methods' => [],  // 無効化
],
```

```php
/**
 * @response {
 *   "data": {"id": 1, "title": "サンプル"}
 * }
 */
public function show(Post $post): PostResource
```

### @response と @responseField の関係

| 方法 | レスポンス例 | 型情報 |
|------|------------|--------|
| `@response` のみ | ✅ | ❌ |
| `@responseField` のみ | ❌ | ✅ |
| 両方記述 | ✅ | ✅ |

Scribeで完全なドキュメントを作成するには、`@response` と `@responseField` の両方が必要です。

---

## 検証内容

以下のサンプルAPIを作成し、両ツールの出力を比較しました。

### サンプルAPI構成

```
app/
├── Http/
│   ├── Controllers/
│   │   ├── PostController.php           # 基本CRUD
│   │   ├── PostDetailController.php     # ネスト構造テスト用
│   │   └── PostSearchController.php     # クエリパラメータテスト用
│   ├── Requests/
│   │   ├── StorePostRequest.php
│   │   ├── UpdatePostRequest.php
│   │   └── SearchPostRequest.php        # 検索用クエリパラメータ
│   └── Resources/
│       ├── PostResource.php
│       ├── PostDetailResource.php       # 複雑なネスト構造
│       └── AuthorResource.php
└── Models/
    └── Post.php
```

### ネスト構造のテスト用Resource

型推論の精度を検証するため、複雑なネスト構造を持つResourceを作成しました。

```php
// PostDetailResource.php
return [
    'id' => $this->id,
    'title' => $this->title,
    'author' => new AuthorResource($this->whenLoaded('author')),  // 別のResource
    'tags' => $this->tags->map(fn ($tag) => [                     // 配列変換
        'id' => $tag->id,
        'name' => $tag->name,
    ]),
    'comments_count' => $this->when(...),                         // 条件付きフィールド
    'metadata' => [                                                // インラインネスト
        'view_count' => $this->view_count,
        'seo' => [
            'meta_title' => $this->meta_title,
        ],
    ],
];
```

---

## 比較結果

### 1. FormRequest（バリデーション）の表示

| 項目 | Scramble | Scribe |
|------|----------|--------|
| **title** | `string`, `<= 255 characters`, `required`（赤バッジ） | `string`, "Must not be greater than 255 characters" |
| **status** | `Allowed values: draft \| published \| archived` | "Must be one of: draft, published, archived" |
| **published_at** | `string<date-time> or null` | `string optional`, "Must be a valid date" |
| **required表示** | 視覚的に分かりやすい | optionalのみ表示 |

**Scrambleの優位点**: OpenAPI標準に準拠した型表示、requiredが視覚的に明確

### 2. ネスト構造の推論精度（Scramble）

PHPDocなしでの自動推論結果：

| フィールド | 期待する型 | Scrambleの推論 | 判定 |
|-----------|-----------|---------------|------|
| `author` | AuthorResource | ✅ `$ref: AuthorResource` | 完璧 |
| `metadata` | object | ✅ `object`（中身も展開） | 完璧 |
| `metadata.seo` | object | ✅ `object`（さらにネスト） | 完璧 |
| `reading_time` | integer | ✅ `integer` | 完璧（戻り値の型宣言から） |
| `tags` | array | ❌ `string` | 失敗 |
| `comments_count` | integer | ❌ `string` | 失敗 |
| `id` | integer | ❌ `string` | 失敗 |

**推論が失敗するパターン：**
- `$this->tags->map()` などのCollection変換
- `$this->when()` の条件付きフィールド
- モデルが見つからない場合の基本フィールド

### 3. ScrambleのPHPDoc補完

推論が失敗した部分はPHPDocで補完できます。

```php
/**
 * @mixin Post  ← クラスレベルでモデルを指定
 */
class PostDetailResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            /** @var int 投稿ID */
            'id' => $this->id,
            
            /**
             * @var array<int, array{id: int, name: string, slug: string}> タグ一覧
             */
            'tags' => $this->tags->map(...),
            
            /**
             * @var int|null コメント数
             */
            'comments_count' => $this->when(...),
        ];
    }
}
```

**注意**: `@return array{...}` 形式は認識されません。インラインPHPDocを使用してください。

### 4. PHPDoc補完後の結果

| フィールド | Before | After |
|-----------|--------|-------|
| `id` | `string` | ✅ `integer` + 説明 |
| `tags` | `string` | ✅ `array` with nested object |
| `comments_count` | `string` | ✅ `integer \| null` |
| `metadata.view_count` | `string` | ✅ `integer` |
| `metadata.is_featured` | `string \| boolean` | ✅ `boolean` |

### 5. 記述量の比較

同じネスト構造のエンドポイントで比較：

#### Scribe（約60行）

```php
/**
 * @responseField data object 投稿データ
 * @responseField data.id integer 投稿ID
 * @responseField data.title string 記事タイトル
 * @responseField data.author object 著者情報
 * @responseField data.author.id integer 著者ID
 * @responseField data.author.name string 著者名
 * @responseField data.tags array タグ一覧
 * @responseField data.tags[].id integer タグID
 * @responseField data.metadata object メタデータ
 * @responseField data.metadata.view_count integer 閲覧数
 * @responseField data.metadata.seo object SEO情報
 * @responseField data.metadata.seo.meta_title string メタタイトル
 * // ... 以下続く
 *
 * @response {
 *   "data": {
 *     "id": 1,
 *     "title": "サンプル記事",
 *     // ... JSONサンプル全体
 *   }
 * }
 */
```

#### Scramble（約15行）

```php
/**
 * @mixin Post
 */
class PostDetailResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            /** @var int 投稿ID */
            'id' => $this->id,
            
            /** @var array<int, array{id: int, name: string, slug: string}> */
            'tags' => $this->tags->map(...),
            
            /** @var int|null */
            'comments_count' => $this->when(...),
            // 他のフィールドは自動推論
        ];
    }
}
```

### 6. クエリパラメータの扱い

GETリクエストでFormRequestを使った検索APIで検証しました。

#### Scramble

```php
class SearchPostRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'keyword' => ['nullable', 'string', 'max:100'],
            'status' => ['nullable', 'string', 'in:draft,published,archived'],
            'per_page' => ['nullable', 'integer', 'min:1', 'max:100'],
        ];
    }
}
```

**結果**: GETリクエストなら自動でQuery Parametersとして認識。バリデーションルールから制約も自動取得。

#### Scribe

デフォルトではFormRequestのルールを**Body Parameters**として扱うため、明示的な指定が必要。

**方法1: FormRequestのDocBlockに「Query parameters」を含める**
```php
/**
 * Query parameters
 */
class SearchPostRequest extends FormRequest
```

**方法2: `queryParameters()` メソッドを追加**
```php
public function queryParameters(): array
{
    return [
        'keyword' => ['description' => '検索キーワード', 'example' => 'Laravel'],
    ];
}
```

**方法3: コントローラーで `@queryParam` を使用**
```php
/**
 * @queryParam keyword string 検索キーワード。Example: Laravel
 */
```

指定しないと、curlコマンドに不要な `--data` が含まれてしまいます：

```bash
# 修正前（Query + Body両方）
curl --get "http://localhost/api/posts/search?keyword=Laravel" \
    --data '{"keyword": "b"}'

# 修正後（Queryのみ）
curl --get "http://localhost/api/posts/search?keyword=Laravel"
```

### 7. 複数レスポンス例

検索APIのように、条件によってレスポンスが異なるケースの対応。

#### Scribe

`@response` の `scenario` パラメータで複数パターンを定義可能：

```php
/**
 * @response status=200 scenario="検索結果あり" {
 *   "data": [
 *     {"id": 1, "title": "Laravelの基本"},
 *     {"id": 2, "title": "Laravel APIの作り方"}
 *   ],
 *   "meta": {"total": 72}
 * }
 *
 * @response status=200 scenario="検索結果なし" {
 *   "data": [],
 *   "meta": {"total": 0}
 * }
 *
 * @response status=422 scenario="バリデーションエラー" {
 *   "message": "The given data was invalid.",
 *   "errors": {"status": ["選択されたステータスは正しくありません。"]}
 * }
 */
```

ドキュメントには「Example response (200, 検索結果あり)」「Example response (200, 検索結果なし)」のように表示されます。

#### Scramble

`#[Response]` アトリビュートの `examples` パラメータで複数のExampleを定義できますが、**UIでは1つ目のみ表示**されます。また、Exampleオブジェクト全体（`value`, `summary`, `description`）がJSON構造として表示されるため、Scribeのようなクリーンなレスポンス例の表示にはなりません。

:::message
内部的にはOpenAPI JSONに複数の `examples` が出力されています。UIの制約（Stoplight Elements）により1つのみ表示される状態です。
:::

```php
#[Response(
    status: 200,
    description: '検索結果',
    examples: [
        'results' => new Example(value: [...], summary: '検索結果あり'),
        'empty' => new Example(value: [...], summary: '検索結果なし'),  // JSONには出力されるがUIでは表示されない
    ]
)]
```

**結論: 複数レスポンス例（シナリオ）の表示は、Scribeが明確に優位**です。

### 8. 生成方式の違い

| 項目 | Scramble | Scribe |
|------|----------|--------|
| 生成方式 | 静的解析（リアルタイム） | コマンド実行 |
| DB接続 | 不要 | response_calls使用時は必要 |
| テストデータ | 不要 | `@response`で回避可能 |
| API数増加時の生成時間 | 影響小 | `response_calls`有効時は増加 |

Scribeで `response_calls` を有効にしている場合、APIエンドポイントが増えるほど実際のHTTPリクエストが増加し、生成時間が長くなります。

**対策:**
- `response_calls` をGETのみに限定
- `response_calls` を無効化し、`@response` で手動定義
- 特定のルートを除外

---

## 両ツールの比較

### 特性の違い

**Scribe**は機能が豊富でドキュメント内容のカスタマイズ性が高い反面、アノテーションの記述量が多くなる傾向があります。また、アノテーションベースのため、コードの型を変更してもコメントを更新し忘れると情報が乖離するリスクがあります。

**Scramble**は、FormRequestやResourceからの型推論でドキュメントを生成するため、シンプルなドキュメントであれば記述コストが低いです。コードを直接解析するため、コードの変更が（推論可能な範囲で）即座にドキュメントに反映されます。ただし、ドキュメントを充実させようとするとPHPDocの追加が必要になり、複数レスポンス例を表示できないなどの制約があります。

### コストと完成度のイメージ

```
ドキュメントの充実度
    ↑
100%│          ┌─────────── Scribe（手動記述で自由度高）
    │         ╱
 80%│────────●  Scramble（自動推論の限界）
    │       ╱│
    │      ╱ │
 20%│─────●  │
    │    ╱   │
  0%│───●────┼──────────────→ 記述コスト
       低    中          高
```

- **Scramble**: 低コストで80%程度の完成度を達成できるが、それ以上は制約に当たる
- **Scribe**: 記述量は増えるが、100%の完成度を目指せる

### 機能比較表

| 機能 | Scramble | Scribe |
|------|----------|--------|
| レスポンス例（JSON） | ⚠️ `#[Response]`で可能だが表示が構造的 | ✅ `@response` でクリーンに表示 |
| 複数レスポンス例 | ❌ UIでは1つのみ | ✅ `scenario` で複数表示・切替 |
| 型情報 | ✅ 自動推論 + PHPDoc | ✅ `@responseField` |
| エラーレスポンス | ✅ 404, 422, 403等を自動検出 | ❌ 手動定義 |
| クエリパラメータ | ✅ GETなら自動認識 | ⚠️ 明示的指定が必要 |
| バリデーション制約 | ✅ 自動（min, max, in等） | ⚠️ 説明文に手動記述 |
| ページネーション | ✅ 詳細スキーマを自動生成 | ✅ 実レスポンスから |
| cURLサンプル | ✅ あり | ✅ あり |
| Try It Out | ✅ あり | ✅ あり |
| Postmanコレクション | ❌ なし | ✅ あり |
| OpenAPIバージョン | 3.1.0 | 3.0.3（設定で3.1.0も可） |
| 生成方式 | リアルタイム（静的解析） | コマンド実行 |

---

## まとめ

### Scrambleの特徴

**強み:**
- 導入が簡単（FormRequest/Resourceがあればほぼゼロコンフィグ）
- 記述量が少ない（コードから自動推論、PHPDocは補完のみ）
- 静的解析ベースでDB接続不要
- エラーレスポンス（404, 422, 403等）を自動検出（`@throws` アノテーションにも対応）
- リアルタイム生成でコマンド実行不要

**制約:**
- 複数レスポンス例はUIで1つのみ表示
- `map()`, `when()` などは型推論が失敗するため手動補完が必要
- Scribeほどの細かいカスタマイズは難しい

### Scribeの特徴

**強み:**
- ドキュメント内容を細かく制御可能
- 複数シナリオ（`scenario`）で条件によるレスポンス違いを明示
- 実際のJSONサンプルをクリーンに表示
- Postmanコレクションの自動生成
- 豊富なドキュメントと実績

**注意点:**
- ネスト構造ではコメント量が多くなりがち
- コードとアノテーションの二重管理（変更時の乖離リスク）
- `response_calls` 有効時は生成時間がAPI数に比例して増加
- 設定によってはDB接続が必要

### 選定の目安

| 状況 | 推奨 | 理由 |
|------|------|------|
| シンプルなドキュメントで十分 | Scramble | 最小限の記述で完成 |
| レスポンス例を詳細に記載したい | Scribe | クリーンなJSON表示 |
| 複数シナリオの表示が必要 | Scribe | `scenario` で対応 |
| 既存のPHPDoc資産がある | Scribe | 資産を活かせる |
| 細部までカスタマイズしたい | Scribe | 自由度が高い |

両ツールは併用も可能なので、プロジェクトの要件に応じて選択してください。

---

## 参考リンク

- [Scramble 公式ドキュメント](https://scramble.dedoc.co/)
- [Scribe 公式ドキュメント](https://scribe.knuckles.wtf/laravel)
