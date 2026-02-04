---
title: "【Laravel × Next.js】Scramble + Orvalで型安全なAPI開発を試してみた"
emoji: "🔗"
type: "tech"
topics: ["laravel", "nextjs", "typescript", "openapi", "reactquery"]
published: true
---

## はじめに

Laravel REST APIとTypeScript製フロントエンドを連携する際、こんな悩みがありました。

- APIのレスポンス型を手動で定義するのが面倒
- バックエンドの変更がフロントエンドに伝わらず、ランタイムエラーになる
- API呼び出しのコードが冗長で、毎回似たような実装を書いている

前回の記事で**Scramble**（Laravel用OpenAPI自動生成）を検証したところ、型安全なフロントエンド開発との相性が良さそうだったので、今回は**Orval**（OpenAPIからTypeScript/React Query生成）と組み合わせて、実際に型安全なAPI開発フローを構築できるか試してみました。

### 今回検証したこと

```
Laravel API → Scramble → OpenAPI JSON → Orval → TypeScript型 + React Queryフック → Next.js
```

バックエンドで型を定義するだけで、フロントエンドの型定義とAPI呼び出しコードが自動生成される開発体験を目指します。

### 検証環境

- PHP 8.4 / Laravel 12
- Node.js 22 / Next.js 15
- Scramble（最新版）
- Orval 8.2.0
- TanStack Query（React Query）5.x

検証に使用したコードは以下のリポジトリで公開しています。

https://github.com/toshiro3/scramble-orval-demo

:::message
本記事は前回の記事「[Laravel APIドキュメント生成ツール比較：Scribe vs Scramble](https://zenn.dev/toshiro3/articles/laravel-api-docs-scribe-vs-scramble)」の実践編です。Scrambleの基本的な特徴については前回の記事をご参照ください。
:::

## ScrambleとOrvalについて

### Scramble

ScrambleはLaravel専用のOpenAPI（Swagger）ドキュメント自動生成ライブラリです。PHPコードの静的解析により、アノテーション不要でOpenAPIを生成してくれます。FormRequestやAPI Resourceから型を自動推論してくれるのが特徴です。

### Orval

OrvalはOpenAPI仕様からTypeScriptコードを自動生成するツールです。React Query / SWR / Vue Query などのフック生成に対応しており、今回はReact Query用のコードを生成してみます。

### 組み合わせると何が嬉しいか

| 期待するメリット | 内容 |
|---------|------|
| 型の一元管理 | LaravelのFormRequest/Resourceで型を定義すれば、フロントエンドの型は自動生成されるはず |
| 変更の自動追従 | APIの変更はOrval再実行で即座にフロントエンドに反映されるはず |
| 開発効率の向上 | React Queryフックが自動生成されるため、API呼び出しコードを書く必要がないはず |

実際にどこまでうまくいくか、検証していきます。

## 環境構築

### ディレクトリ構成

最終的なディレクトリ構成は以下のようになりました。

```
scramble-orval-demo/
├── docker-compose.yml
├── docker/
│   └── php/
│       └── Dockerfile
├── src/              # Laravel（バックエンド）
└── frontend/         # Next.js（フロントエンド）
```

### Laravel環境構築（Docker Compose）

PHPの依存関係管理が面倒なので、Docker Composeで環境を構築しました。

#### Dockerfile

`docker/php/Dockerfile`:

```dockerfile
FROM php:8.4-cli

RUN apt-get update && apt-get install -y \
    git \
    unzip \
    libzip-dev \
    && docker-php-ext-install zip pdo pdo_mysql \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

EXPOSE 8000

CMD ["php", "artisan", "serve", "--host=0.0.0.0", "--port=8000"]
```

:::message alert
最新のLaravel 12はPHP 8.4が必要でした。PHP 8.3だと `composer create-project` 時点ではエラーにならないものの、起動時にバージョンエラーになるので注意が必要です。
:::

#### docker-compose.yml

```yaml
services:
  app:
    build:
      context: ./docker/php
      dockerfile: Dockerfile
    volumes:
      - ./src:/var/www/html
    ports:
      - "8000:8000"
    environment:
      - DB_CONNECTION=sqlite
      - DB_DATABASE=/var/www/html/database/database.sqlite
```

検証用なのでDBはSQLiteを使いました。

#### Laravelプロジェクト作成とScrambleインストール

```bash
# プロジェクト作成
docker run --rm -v $(pwd)/src:/app composer create-project laravel/laravel .

# SQLite用の空ファイル作成
touch src/database/database.sqlite

# Scrambleインストール
docker run --rm -v $(pwd)/src:/app -w /app composer require dedoc/scramble
```

起動してScrambleのドキュメント画面が表示されることを確認しました。

```bash
docker compose up -d
docker compose exec app php artisan migrate
```

http://localhost:8000/docs/api にアクセスすると、Scrambleの画面が表示されました。

### Task CRUD APIの作成

検証用のシンプルなTask APIを作成しました。

#### ファイル生成

```bash
docker compose exec app php artisan make:model Task -mcrR
docker compose exec app php artisan install:api
```

Laravel 11以降では `routes/api.php` がデフォルトで存在しないため、`install:api` コマンドが必要でした。これを知らずにしばらくハマりました。

#### Migration

`src/database/migrations/xxxx_create_tasks_table.php`:

```php
public function up(): void
{
    Schema::create('tasks', function (Blueprint $table) {
        $table->id();
        $table->string('title');
        $table->text('description')->nullable();
        $table->enum('status', ['pending', 'in_progress', 'completed'])->default('pending');
        $table->date('due_date')->nullable();
        $table->timestamps();
    });
}
```

:::message
MySQLを使用する場合、`$table->timestamps()` は TIMESTAMP 型になります。TIMESTAMP 型には2038年問題（1970-2038年の範囲制限）とタイムゾーン自動変換の問題があるため、`$table->datetimes()` の使用を検討してください。本記事ではSQLiteを使用しているため、この問題は発生しません。
:::

#### Model

`src/app/Models/Task.php`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Task extends Model
{
    use HasFactory;

    protected $fillable = [
        'title',
        'description',
        'status',
        'due_date',
    ];

    protected $casts = [
        'due_date' => 'date',
    ];
}
```

#### FormRequest

`src/app/Http/Requests/StoreTaskRequest.php`:

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StoreTaskRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'description' => ['nullable', 'string'],
            'status' => ['nullable', 'in:pending,in_progress,completed'],
            'due_date' => ['nullable', 'date'],
        ];
    }
}
```

`src/app/Http/Requests/UpdateTaskRequest.php`:

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class UpdateTaskRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'title' => ['sometimes', 'required', 'string', 'max:255'],
            'description' => ['nullable', 'string'],
            'status' => ['nullable', 'in:pending,in_progress,completed'],
            'due_date' => ['nullable', 'date'],
        ];
    }
}
```

#### API Resource

`src/app/Http/Resources/TaskResource.php`:

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class TaskResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'description' => $this->description,
            'status' => $this->status,
            'due_date' => $this->due_date?->format('Y-m-d'),
            'created_at' => $this->created_at->toISOString(),
            'updated_at' => $this->updated_at->toISOString(),
        ];
    }
}
```

#### Controller

`src/app/Http/Controllers/TaskController.php`:

```php
<?php

namespace App\Http\Controllers;

use App\Http\Requests\StoreTaskRequest;
use App\Http\Requests\UpdateTaskRequest;
use App\Http\Resources\TaskResource;
use App\Models\Task;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;

class TaskController extends Controller
{
    /**
     * タスク一覧を取得
     */
    public function index(): AnonymousResourceCollection
    {
        $tasks = Task::orderBy('created_at', 'desc')->get();

        return TaskResource::collection($tasks);
    }

    /**
     * タスクを作成
     */
    public function store(StoreTaskRequest $request): TaskResource
    {
        $task = Task::create($request->validated());

        return new TaskResource($task);
    }

    /**
     * タスク詳細を取得
     */
    public function show(Task $task): TaskResource
    {
        return new TaskResource($task);
    }

    /**
     * タスクを更新
     */
    public function update(UpdateTaskRequest $request, Task $task): TaskResource
    {
        $task->update($request->validated());

        return new TaskResource($task);
    }

    /**
     * タスクを削除
     */
    public function destroy(Task $task): \Illuminate\Http\JsonResponse
    {
        $task->delete();

        return response()->json(null, 204);
    }
}
```

#### ルート定義

`src/routes/api.php`:

```php
<?php

use App\Http\Controllers\TaskController;
use Illuminate\Support\Facades\Route;

Route::apiResource('tasks', TaskController::class);
```

マイグレーションを実行して、ルートが登録されていることを確認しました。

```bash
docker compose exec app php artisan migrate

docker compose exec app php artisan route:list
# api/tasks のルートが表示されればOK
```

### OpenAPI JSONの確認

http://localhost:8000/docs/api.json にアクセスすると、Scrambleが生成したOpenAPI JSONを確認できました。

```json
{
  "openapi": "3.1.0",
  "components": {
    "schemas": {
      "TaskResource": {
        "type": "object",
        "properties": {
          "id": { "type": "integer" },
          "title": { "type": "string" },
          "description": { "type": ["string", "null"] },
          "status": { "type": "string" },
          "due_date": { "type": "string" },
          "created_at": { "type": "string" },
          "updated_at": { "type": "string" }
        },
        "required": ["id", "title", "description", "status", "due_date", "created_at", "updated_at"]
      },
      "StoreTaskRequest": {
        "type": "object",
        "properties": {
          "title": { "type": "string", "maxLength": 255 },
          "description": { "type": ["string", "null"] },
          "status": { "type": ["string", "null"], "enum": ["pending", "in_progress", "completed"] },
          "due_date": { "type": ["string", "null"], "format": "date-time" }
        },
        "required": ["title"]
      }
    }
  }
}
```

確認できたポイント：
- FormRequestの `rules()` からリクエストの型が推論されている
- API Resourceの `toArray()` からレスポンスの型が推論されている
- `status` のバリデーション `in:pending,in_progress,completed` が `enum` として認識されている
- `nullable` なフィールドは `type: ["string", "null"]` として表現されている

アノテーションなしでここまで推論してくれるのは、前回の検証で確認した通りですが、改めて便利だと感じました。

## Next.js + Orval環境構築

### Next.jsプロジェクト作成

```bash
npx create-next-app@latest frontend
```

最新版ではデフォルト設定（TypeScript, ESLint, Tailwind CSS, App Router）をまとめて選択できるようになっていました。

### パッケージインストール

```bash
cd frontend

npm install orval --save-dev
npm install @tanstack/react-query
npm install axios
```

:::message alert
Orval 8.x は Node.js 22.18.0 以上が必要です。Node.js 20系だと実行時にエラーになりました。Voltaを使っている場合は以下のコマンドでバージョンを固定できます。

```bash
volta pin node@22
```
:::

### Orval設定ファイル作成

`frontend/orval.config.ts`:

```typescript
import { defineConfig } from 'orval';

export default defineConfig({
  api: {
    input: {
      target: 'http://localhost:8000/docs/api.json',
    },
    output: {
      mode: 'tags-split',
      target: './lib/api/generated',
      schemas: './lib/api/generated/schemas',
      client: 'react-query',
      httpClient: 'axios',
      prettier: true,
      override: {
        mutator: {
          path: './lib/api/axios.ts',
          name: 'customInstance',
        },
      },
    },
  },
});
```

設定の意味：

| オプション | 説明 |
|-----------|------|
| `input.target` | OpenAPI JSONのURL。Scrambleのエンドポイントを指定 |
| `mode: 'tags-split'` | OpenAPIのタグごとにファイルを分割 |
| `client: 'react-query'` | TanStack Query用のフックを生成 |
| `httpClient: 'axios'` | HTTPクライアントとしてAxiosを使用 |
| `override.mutator` | カスタムAxiosインスタンスを指定（baseURL設定に必要） |

### カスタムAxiosインスタンス作成

`frontend/lib/api/axios.ts`:

```typescript
import Axios, { AxiosRequestConfig } from 'axios';

const axios = Axios.create({
  baseURL: 'http://localhost:8000/api',
  headers: {
    'Content-Type': 'application/json',
  },
});

export const customInstance = async <T>(
  config: AxiosRequestConfig
): Promise<T> => {
  const response = await axios(config);
  return response.data;
};
```

最初 `export default` で定義したところ、生成されたコードで `return default<TasksIndex200>(...)` のようになってしまい構文エラーになりました。`export const customInstance` のように名前付きエクスポートにする必要がありました。

### コード生成実行

```bash
npx orval
```

```
🍻 orval v8.2.0 - A swagger client generator for typescript
🎉 api - Your OpenAPI spec has been converted into ready to use orval!
```

生成成功です。

## 生成されたコードの確認

### ディレクトリ構成

```
frontend/lib/api/generated/
├── schemas/
│   ├── index.ts
│   ├── taskResource.ts
│   ├── storeTaskRequest.ts
│   ├── storeTaskRequestStatus.ts
│   └── ...
└── task/
    └── task.ts
```

タグごとにディレクトリが分かれ、スキーマは `schemas/` にまとめられていました。

### 型定義（schemas）

`frontend/lib/api/generated/schemas/taskResource.ts`:

```typescript
export interface TaskResource {
  id: number;
  title: string;
  /** @nullable */
  description: string | null;
  status: string;
  due_date: string;
  created_at: string;
  updated_at: string;
}
```

`frontend/lib/api/generated/schemas/storeTaskRequest.ts`:

```typescript
import type { StoreTaskRequestStatus } from './storeTaskRequestStatus';

export interface StoreTaskRequest {
  /** @maxLength 255 */
  title: string;
  /** @nullable */
  description?: string | null;
  /** @nullable */
  status?: StoreTaskRequestStatus;
  /** @nullable */
  due_date?: string | null;
}
```

`frontend/lib/api/generated/schemas/storeTaskRequestStatus.ts`:

```typescript
export type StoreTaskRequestStatus = typeof StoreTaskRequestStatus[keyof typeof StoreTaskRequestStatus] | null;

export const StoreTaskRequestStatus = {
  pending: 'pending',
  in_progress: 'in_progress',
  completed: 'completed',
} as const;
```

確認できたポイント：
- `description: string | null` - nullableなフィールドはunion型で表現されている
- `title: string` - requiredなフィールドはoptionalになっていない
- `StoreTaskRequestStatus` - enumは `as const` オブジェクトとして生成されている（TypeScriptのベストプラクティス）

期待通りの型が生成されていました。

### React Queryフック

`frontend/lib/api/generated/task/task.ts` には以下のフックが生成されていました。

| フック | 用途 | 種別 |
|--------|------|------|
| `useTasksIndex` | タスク一覧取得 | useQuery |
| `useTasksStore` | タスク作成 | useMutation |
| `useTasksShow` | タスク詳細取得 | useQuery |
| `useTasksUpdate` | タスク更新 | useMutation |
| `useTasksDestroy` | タスク削除 | useMutation |

GETは `useQuery`、POST/PUT/DELETEは `useMutation` として生成されていて、React Queryのベストプラクティスに沿っています。

## 実際のAPI呼び出し

生成されたフックを使って、実際にタスクを表示・作成してみました。

### React Query Provider設定

`frontend/app/providers.tsx`:

```tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useState } from 'react';

export default function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient());

  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
}
```

`frontend/app/layout.tsx` にProviderを追加：

```tsx
import Providers from "./providers";

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="en">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

### タスク一覧ページ

`frontend/app/page.tsx`:

```tsx
'use client';

import { useTasksIndex, useTasksStore } from '@/lib/api/generated/task/task';
import { useState } from 'react';

export default function Home() {
  const [title, setTitle] = useState('');

  // タスク一覧取得
  const { data: tasks, isLoading, error, refetch } = useTasksIndex();

  // タスク作成
  const createTask = useTasksStore({
    mutation: {
      onSuccess: () => {
        setTitle('');
        refetch();
      },
    },
  });

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (!title.trim()) return;
    createTask.mutate({ data: { title } });
  };

  if (isLoading) return <div className="p-8">Loading...</div>;
  if (error) return <div className="p-8 text-red-500">Error: {String(error)}</div>;

  return (
    <main className="p-8 max-w-2xl mx-auto">
      <h1 className="text-2xl font-bold mb-6">Task List</h1>

      <form onSubmit={handleSubmit} className="mb-6 flex gap-2">
        <input
          type="text"
          value={title}
          onChange={(e) => setTitle(e.target.value)}
          placeholder="New task title"
          className="flex-1 border rounded px-3 py-2 bg-white text-black"
        />
        <button
          type="submit"
          disabled={createTask.isPending}
          className="bg-blue-500 text-white px-4 py-2 rounded disabled:opacity-50"
        >
          {createTask.isPending ? 'Adding...' : 'Add Task'}
        </button>
      </form>

      <ul className="space-y-2">
        {tasks?.data?.map((task) => (
          <li
            key={task.id}
            className="border rounded p-3 flex justify-between items-center"
          >
            <span>{task.title}</span>
            <span className="text-sm text-gray-500">{task.status}</span>
          </li>
        ))}
        {tasks?.data?.length === 0 && (
          <li className="text-gray-500">No tasks yet</li>
        )}
      </ul>
    </main>
  );
}
```

確認できたポイント：
- `useTasksIndex()` を呼ぶだけでタスク一覧が取得できる
- `createTask.mutate({ data: { title } })` の引数は型チェックされる
- `tasks?.data?.map((task) => ...)` の `task` は `TaskResource` 型として認識される

タスクの作成・一覧表示が問題なく動作しました。

## 型安全の検証

ここからが今回の検証の**本題**です。バックエンドの型を変更したときに、フロントエンド側でどう検知されるか確認しました。

### シナリオ: TaskResourceに新しいフィールドを追加

Laravel側で `priority` フィールドを追加して、フロントエンドへの影響を見てみます。

### Laravel側の変更

Migration:

```bash
docker compose exec app php artisan make:migration add_priority_to_tasks_table
```

```php
public function up(): void
{
    Schema::table('tasks', function (Blueprint $table) {
        $table->enum('priority', ['low', 'medium', 'high'])->default('medium')->after('status');
    });
}
```

Model、TaskResource、StoreTaskRequestにも `priority` を追加しました。

### Orval再実行

```bash
npx orval
```

### 型定義の変更を確認

`frontend/lib/api/generated/schemas/taskResource.ts`:

```typescript
export interface TaskResource {
  id: number;
  title: string;
  /** @nullable */
  description: string | null;
  status: string;
  priority: string;  // ← 自動的に追加された
  due_date: string;
  created_at: string;
  updated_at: string;
}
```

バックエンドで追加した `priority` フィールドが、フロントエンドの型定義に反映されていました。

### フロントエンドで新しいフィールドを使用

```tsx
{tasks?.data?.map((task) => (
  <li key={task.id}>
    <span>{task.title}</span>
    <span>{task.priority}</span>  {/* IDEの補完で候補に出る */}
  </li>
))}
```

VSCodeの補完で `task.priority` が候補に出ることを確認しました。

### 存在しないプロパティへのアクセスは型エラー

試しにタイポしてみました。

```tsx
<span>{task.priorityyyy}</span>
```

即座にTypeScriptエラーが表示されました。

```
Property 'priorityyyy' does not exist on type 'TaskResource'.
```

**これが型安全の威力です**。ランタイムエラーになる前に、開発時点（コンパイル時）でエラーを検知できます。

### 型チェックコマンド

CIで型の整合性をチェックする場合は以下のコマンドが使えます。

```bash
npx tsc --noEmit
```

バックエンドの変更によってフロントエンドの型が壊れた場合、このコマンドで検知できます。

## 開発フローの自動化

### package.json にスクリプト追加

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "generate:api": "orval"
  }
}
```

:::message alert
**「鶏と卵」問題に注意**: `npx orval` を実行する際、Laravelのサーバーが起動している必要があります。`predev` に `generate:api` を入れると、初回起動時にAPIサーバーが動いておらずJSONが取得できないエラーになりがちです。

**対策案**:
1. 開発時は手動で `npm run generate:api` を実行する
2. OpenAPI JSONをローカルファイルとして書き出し、`input.target` にローカルパスを指定する
3. `wait-on` パッケージを使ってAPIサーバーの起動を待つ
:::

ローカルファイルを使う場合の設定例：

```typescript
// orval.config.ts
export default defineConfig({
  api: {
    input: {
      target: './openapi.json',  // ローカルファイルを指定
    },
    // ...
  },
});
```

```bash
# OpenAPI JSONをローカルに書き出す
curl http://localhost:8000/docs/api.json > openapi.json
```

### CI/CD での活用

GitHub Actionsなどで以下のようなワークフローを組むと、PRマージ前に型の整合性をチェックできます。

```yaml
- name: Generate API types
  run: npm run generate:api

- name: Type check
  run: npx tsc --noEmit
```

## 検証結果まとめ

### 確認できたこと

| 項目 | 結果 |
|------|------|
| ScrambleによるOpenAPI自動生成 | ✅ FormRequest、API Resourceから型が正しく推論された |
| OrvalによるTypeScript型生成 | ✅ nullable、enum含め正しい型が生成された |
| React Queryフック生成 | ✅ GET→useQuery、POST/PUT/DELETE→useMutationで生成された |
| バックエンド変更の追従 | ✅ Orval再実行で型定義に反映された |
| 型エラーの検知 | ✅ 存在しないプロパティへのアクセスはコンパイル時にエラー |

### メリット

| メリット | 説明 |
|---------|------|
| **型の一元管理** | LaravelのFormRequest/Resourceで型を定義するだけ。フロントエンドの型は自動生成される |
| **開発効率の向上** | API呼び出しコード、型定義を手書きする必要がない |
| **型安全の保証** | 存在しないプロパティへのアクセスはコンパイル時にエラー |
| **変更の自動追従** | バックエンドの変更はOrval再実行で即座に反映 |
| **IDEサポート** | 補完、リファクタリング、ジャンプが効く |

### 注意点・ハマったポイント

| 注意点 | 内容 |
|-------|------|
| **Node.js バージョン** | Orval 8.x は Node.js 22以上が必要。20系だと実行時エラー |
| **Laravel 11以降のAPI ルート** | `routes/api.php` がデフォルトで存在しない。`php artisan install:api` が必要 |
| **カスタムAxiosの書き方** | `export default` だと生成コードが壊れる。名前付きエクスポートが必要 |
| **CORSの設定** | 開発環境ではデフォルトで動作。本番環境では適切に設定（後述） |
| **OpenAPI生成の精度** | 複雑なレスポンスは正確に推論されない場合がある（後述） |

#### CORS設定について

Laravel 11以降では、`HandleCors` ミドルウェアがデフォルトで有効になっており、**デフォルト設定で `allowed_origins => ['*']`（全オリジン許可）** になっています。そのため、開発環境では追加設定なしでフロントエンド（localhost:3000）からバックエンド（localhost:8000）へのリクエストが動作します。

本番環境でオリジンを制限したい場合は、以下のコマンドで設定ファイルを公開してカスタマイズできます：

```bash
php artisan config:publish cors
```

`config/cors.php` が作成されるので、`allowed_origins` を適切に設定してください：

```php
'allowed_origins' => ['https://your-frontend-domain.com'],
```

#### Scrambleの推論がうまくいかない場合

Scrambleは非常に強力ですが、複雑なクロージャや動的な条件分岐がある場合、型推論が `any` や `unknown` になることがあります。その場合は、**インラインPHPDoc**で明示的に型を指定することで補完できます。

```php
/**
 * @mixin Post  // クラスレベルでモデルを指定
 */
class PostDetailResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            /** @var int 投稿ID */
            'id' => $this->id,

            /** @var string 記事タイトル */
            'title' => $this->title,

            /**
             * @var array<int, array{id: int, name: string}> タグ一覧
             */
            'tags' => $this->tags->map(fn ($tag) => [
                'id' => $tag->id,
                'name' => $tag->name,
            ]),

            /** @var int|null コメント数 */
            'comments_count' => $this->when($this->comments_count !== null, $this->comments_count),
        ];
    }
}
```

:::message alert
**注意**: `@return array{...}` 形式はScrambleでは認識されません。各フィールドの直前にインラインPHPDoc（`/** @var ... */`）を記述してください。
:::

### この開発フローが向いているケース

- Laravel + TypeScript製フロントエンドの組み合わせ
- APIの型定義を手動管理したくない
- チーム開発で型の整合性を保証したい
- React Query（TanStack Query）を使用している

## おわりに

Scramble + Orvalの組み合わせで、期待通りの型安全なAPI開発フローを構築できました。

バックエンドで型を定義すれば、フロントエンドの型定義とAPI呼び出しコードが自動生成される。この開発体験は一度味わうと手放せなくなりそうです。

特にチーム開発では、APIの変更がフロントエンドの型エラーとして検知できるのは大きなメリットだと感じました。

## 参考リンク

- [Scramble - Laravel API Documentation Generator](https://scramble.dedoc.co/)
- [Orval - Restful client generator](https://orval.dev/)
- [TanStack Query](https://tanstack.com/query)
- [前回の記事：Laravel APIドキュメント生成ツール比較：Scribe vs Scramble](https://zenn.dev/toshiro3/articles/laravel-api-docs-scribe-vs-scramble)
