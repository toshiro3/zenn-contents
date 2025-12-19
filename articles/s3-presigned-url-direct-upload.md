---
title: "S3 Presigned URLを使った直接アップロードでAPIパフォーマンスを改善する"
emoji: "🚀"
type: "tech"
topics: ["aws", "s3", "laravel", "php"]
published: true
---

## はじめに

Webアプリケーションにおいて、画像アップロードのパフォーマンスはユーザー体験に直結します。本記事では、LaravelバックエンドでBase64画像の同期アップロードがボトルネックとなっていた問題を、**S3 Presigned URL**を使ったフロントエンドからの直接アップロード方式に変更することで解決した事例を紹介します。

## 背景

### 改善前のアーキテクチャ

フロントエンドで生成した画像をBase64エンコードしてAPIに送信し、バックエンドでS3にアップロードする構成でした。

```
[フロントエンド]
    ↓ POST /api/upload
    ↓ (Base64エンコードされた画像を含む)
[Controller]
    ↓
[画像保存処理]
    ├─ メタ情報をDBに保存
    ├─ 画像ごとにループ:
    │   ├─ Base64デコード
    │   ├─ Storage::put() でS3に同期保存 ← ★ボトルネック
    │   └─ DBのimage_pathを更新
    └─ 後続処理のJob dispatch (非同期)
    ↓
[レスポンス返却]
```

### 問題点

| 問題 | 影響 |
|------|------|
| `Storage::put()` が同期処理 | S3アップロード完了までリクエストがブロック |
| 複数画像の直列処理 | 待ち時間が積み重なる |
| ネットワーク状況に依存 | 1秒以上の遅延が発生しうる |

## 解決策：Presigned URLによる直接アップロード

### 改善後のアーキテクチャ

```
[フロントエンド]
    ↓ POST /api/upload/prepare
    ↓ (画像データなし、メタ情報のみ)
[Controller]
    ↓ Presigned URL発行
    ↓
[レスポンス返却（高速）]
    ↓
[フロントエンド]
    ├─ Presigned URLを使ってS3に直接PUT ← 並列実行可能
    ↓
    ↓ POST /api/upload/complete
[Controller]
    ├─ ファイル存在確認
    ├─ DB更新
    └─ 後続処理のJob dispatch
```

### 期待される効果

- **APIレスポンス時間の大幅短縮**: バックエンドでのBase64デコード・S3アップロードが不要
- **並列アップロード**: 複数画像を同時にアップロード可能
- **スケーラビリティ向上**: バックエンドサーバーの負荷軽減

## 実装

### 1. ルーティング

```php
// routes/api.php

// 新規作成 + Presigned URL発行
Route::post('/upload/prepare', 'UploadController@prepare');

// 更新 + Presigned URL発行
Route::match(['put', 'patch'], '/upload/prepare/{id}', 'UploadController@prepareUpdate');

// アップロード完了通知
Route::post('/upload/complete', 'UploadController@complete');
```

### 2. Presigned URL発行

```php
use Aws\S3\S3Client;

class UploadController extends Controller
{
    /**
     * S3アップロード用のPresigned URLを発行
     */
    private function getPresignedUrl(string $filePath): string
    {
        $s3Client = new S3Client([
            'version' => 'latest',
            'region'  => config('filesystems.disks.s3.region'),
            'credentials' => [
                'key'    => config('filesystems.disks.s3.key'),
                'secret' => config('filesystems.disks.s3.secret'),
            ],
        ]);

        $command = $s3Client->getCommand('PutObject', [
            'Bucket' => config('filesystems.disks.s3.bucket'),
            'Key'    => $filePath,
            'ContentType' => 'image/png',
        ]);

        $request = $s3Client->createPresignedRequest($command, '+5 minutes');

        return (string) $request->getUri();
    }

    public function prepare(Request $request): JsonResponse
    {
        // バリデーション・DB保存処理...

        $uploadTargets = [];
        foreach ($items as $item) {
            $filePath = "uploads/{$projectId}/image_{$item->id}.png";

            $uploadTargets[] = [
                'itemId'       => $item->id,
                'presignedUrl' => $this->getPresignedUrl($filePath),
                'filePath'     => $filePath,
                'expiresAt'    => now()->addMinutes(5)->format('Y-m-d H:i:s'),
            ];
        }

        return response()->json([
            'projectId'     => $projectId,
            'uploadTargets' => $uploadTargets,
        ]);
    }
}
```

### 3. アップロード完了通知API

ここで重要なのは**セキュリティ検証**です。

```php
public function complete(UploadCompleteRequest $request): JsonResponse
{
    foreach ($request->validated('uploads') as $upload) {
        $itemId   = $upload['itemId'];
        $filePath = $upload['filePath'];

        // ★ ファイルパスからIDを抽出して検証
        $pathInfo = $this->extractIdsFromFilePath($filePath);

        if ($pathInfo === null) {
            abort(400, '無効なファイルパス形式です。');
        }

        // パスのIDとリクエストのIDが一致するか検証
        if ($pathInfo['item_id'] !== $itemId) {
            abort(400, 'ファイルパスとIDが一致しません。');
        }

        // DBからアイテムを取得
        $item = UploadItem::with('project')->findOrFail($itemId);

        // パスのproject_idとDBのproject_idが一致するか検証
        if ($pathInfo['project_id'] !== $item->project_id) {
            abort(400, 'ファイルパスとIDが一致しません。');
        }

        // 所有者チェック
        if ($item->project->user_id !== auth()->id()) {
            abort(403);
        }

        // ファイル存在確認
        if (!Storage::exists($filePath)) {
            Log::warning('アップロードファイルが見つかりません', [
                'item_id'   => $itemId,
                'file_path' => $filePath,
            ]);
            continue;
        }

        // DB更新
        $item->update(['image_path' => $filePath]);

        // 後続処理のJob dispatch
        ProcessUploadedFileJob::dispatch($itemId, $filePath);
    }

    return response()->json(['success' => true]);
}

/**
 * ファイルパスからIDを抽出
 *
 * 期待形式: uploads/{project_id}/image_{item_id}.png
 */
private function extractIdsFromFilePath(string $filePath): ?array
{
    $pattern = '/^uploads\/(\d+)\/image_(\d+)\.png$/';

    if (!preg_match($pattern, $filePath, $matches)) {
        return null;
    }

    return [
        'project_id' => (int) $matches[1],
        'item_id'    => (int) $matches[2],
    ];
}
```

### 4. セキュリティ考慮点

#### AWSがPresigned URLを推奨する理由

Presigned URLを使った直接アップロードは[AWSが推奨する方法](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)です。[AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/presigned-url-best-practices/overview.html)では、以下の観点からPresigned URLの利点が説明されています。

**1. S3のスケーラビリティを直接活用できる**

バックエンドを経由せずS3と直接通信することで、S3の組み込みスケーラビリティを活用できます。API経由の場合、バックエンドがバイトを中継する必要があり、負荷に応じたスケーリングが課題になります。

**2. AWS STSの一時認証情報と比較した利点**

代替手段としてAWS STS（Security Token Service）で一時的な認証情報を発行する方法がありますが、Presigned URLには以下の利点があります：

| 項目 | Presigned URL | AWS STS一時認証情報 |
|------|---------------|---------------------|
| APIコール制限 | なし（署名はローカル生成） | あり（スロットリングの可能性） |
| 最小有効期限 | 実質的に60秒程度から | 15分 |
| レイテンシ | 低（APIコール不要） | 高（STSへのAPIコール必要） |

**3. 最小権限の原則との整合性**

Presigned URLは、発行者に付与された権限を超えるアクセスを提供できません。つまり、IAMポリシーで適切に権限を制限していれば、Presigned URLもその制限を継承します。

#### AWSが挙げている注意点

一方で、AWSのドキュメントでは以下の注意点も挙げられています。

**Bearer Token（持参人トークン）としての性質**

> The capabilities of a presigned URL are limited by the permissions of the user who created it. In essence, presigned URLs are bearer tokens that grant access to those who possess them. As such, we recommend that you protect them appropriately.
>
> （訳：Presigned URLの機能は、作成したユーザーの権限によって制限されます。本質的に、Presigned URLは所持者にアクセスを許可するBearer Tokenです。そのため、適切に保護することを推奨します。）

URLを知っている人は誰でもアクセスできるため、漏洩リスクを考慮した設計が必要です。

**Presigned URLの生成自体は制限できない**

> The process of creating a presigned URL is an algorithmic operation that's based on a published standard (Signature Version 4) for signature generation. Therefore, it's not possible to place limits on the generation of presigned URLs.
>
> （訳：Presigned URLの作成は、公開された標準（Signature Version 4）に基づくアルゴリズム的な操作です。そのため、Presigned URLの生成自体に制限をかけることはできません。）

生成自体は制限できませんが、URLが有効かどうかはリクエスト時に評価されるため、IAMポリシーやバケットポリシーでの制限が重要になります。

#### この実装で対処できていること

フロントエンドがブラウザの場合、悪意のあるユーザーによる攻撃を考慮する必要があります。本実装では以下の対策を行っています。

| 対策 | 目的 | 実装箇所 |
|------|------|----------|
| ファイルパス形式の検証 | 不正なパス形式を拒否 | `extractIdsFromFilePath()` |
| パス内IDとリクエストIDの一致検証 | 改ざんされたIDを検出 | `complete()` |
| 所有者チェック | 他ユーザーのリソースへのアクセス防止 | `complete()` |
| Presigned URLの有効期限 | 漏洩時の影響を限定 | 5分で失効 |
| サーバー側でファイルパスを生成 | パストラバーサル攻撃を防止 | `prepare()` |

**ファイルパス検証が必要な理由**

Presigned URL発行時に `itemId: 100` に対して発行されたURLで、完了通知時に `itemId: 50` を送信されると、意図しないレコードが更新される可能性があります。ファイルパスに埋め込まれたIDと照合することで、この攻撃を防ぎます。

#### この実装では対処が難しいこと

以下のリスクは、追加の対策が必要です。

**1. アップロードされるファイルの内容**

Presigned URLを取得したユーザーは、そのURLに対して任意のファイルをアップロードできます。

```bash
# 攻撃例：画像ではなく実行ファイルをアップロード
curl -X PUT "https://bucket.s3.amazonaws.com/..." \
  -H "Content-Type: image/png" \
  --data-binary @malware.exe
```

Content-Typeヘッダーはクライアント側で設定するため、実際のファイル内容と異なる可能性があります。

**対策例：**
- 完了通知時にファイルのMIMEタイプを検証
- Lambda@Edgeでアップロード時に検証
- 後続のJobでファイル内容を検証し、不正なら削除

```php
// 完了通知時にMIMEタイプを検証する例
$mimeType = Storage::mimeType($filePath);
if (!in_array($mimeType, ['image/png', 'image/jpeg'], true)) {
    Storage::delete($filePath);
    abort(400, '許可されていないファイル形式です。');
}
```

**2. ファイルサイズの制限**

Presigned URL発行時に条件を指定しないと、巨大なファイルをアップロードされる可能性があります。

**対策例：** Presigned URL発行時にContent-Length条件を追加

```php
$command = $s3Client->getCommand('PutObject', [
    'Bucket' => config('filesystems.disks.s3.bucket'),
    'Key'    => $filePath,
    'ContentType' => 'image/png',
]);

// アップロード条件を追加
$request = $s3Client->createPresignedRequest($command, '+5 minutes', [
    // S3のPutObjectでは直接サイズ制限はできないため、
    // S3バケットポリシーまたはLambda@Edgeで制限する
]);
```

S3単体ではPresigned URLにファイルサイズ制限を付けられないため、以下の方法を検討してください：
- フロントエンドでアップロード前にサイズチェック（悪意あるユーザーには効果なし）
- S3 Object Lambdaでサイズ検証
- 完了通知時にサイズを確認し、超過していれば削除

**3. Presigned URLの漏洩・再利用**

URLを知っている人は誰でもアップロードできます。有効期限（5分）で軽減されますが、その間は再利用可能です。

**対策例：**
- 有効期限をさらに短くする（1〜2分）
- 完了通知後にファイルのタイムスタンプを確認し、古すぎるものは拒否
- 同一URLでの複数回アップロードを検知（ただし実装は複雑）

**4. 完了通知の重複呼び出し**

悪意のあるユーザーが完了通知APIを複数回呼び出す可能性があります。

**対策例：** 冪等性を確保する

```php
// すでにimage_pathが設定されていれば処理をスキップ
if ($item->image_path === $filePath) {
    continue; // すでに処理済み
}
```

#### S3バケット側の設定

サーバーサイドの実装だけでなく、S3バケットの設定も重要です。

```json
// バケットポリシー例：特定のContent-Typeのみ許可
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::your-bucket/uploads/*",
      "Condition": {
        "StringNotEquals": {
          "s3:content-type": ["image/png", "image/jpeg"]
        }
      }
    }
  ]
}
```

**CORS設定：** 必要なオリジンのみ許可

```json
[
  {
    "AllowedOrigins": ["https://your-domain.com"],
    "AllowedMethods": ["PUT"],
    "AllowedHeaders": ["Content-Type"],
    "MaxAgeSeconds": 3000
  }
]
```

#### セキュリティのトレードオフ

| アプローチ | メリット | デメリット |
|------------|----------|------------|
| API経由アップロード | サーバーで完全に制御可能 | パフォーマンス低下、サーバー負荷増 |
| Presigned URL | 高速、サーバー負荷低 | ファイル内容の検証が後処理になる |
| Presigned URL + Lambda@Edge | リアルタイム検証可能 | 構成が複雑、コスト増 |

今回の実装は「Presigned URL + 完了通知時の検証」の構成です。悪意のあるファイルがアップロードされる可能性はありますが、以下の条件下では許容できると判断しました：

- アップロードされたファイルは認証済みユーザーの所有物として扱われる
- ファイルは後続処理（Job）でさらに検証・変換される
- 万が一不正なファイルがあっても、影響範囲は当該ユーザーに限定される

### 5. フロントエンド実装（参考）

```typescript
async function uploadFiles(projectData: ProjectData) {
  // 1. Presigned URL取得
  const response = await fetch('/api/upload/prepare', {
    method: 'POST',
    body: JSON.stringify(projectData),
  });
  const { projectId, uploadTargets } = await response.json();

  // 2. S3に並列アップロード
  await Promise.all(
    uploadTargets.map(async (target) => {
      const imageBlob = await generateImage(target.itemId);
      await fetch(target.presignedUrl, {
        method: 'PUT',
        body: imageBlob,
        headers: { 'Content-Type': 'image/png' },
      });
    })
  );

  // 3. 完了通知
  await fetch('/api/upload/complete', {
    method: 'POST',
    body: JSON.stringify({
      uploads: uploadTargets.map((t) => ({
        itemId: t.itemId,
        filePath: t.filePath,
      })),
    }),
  });
}
```

## テスト

```php
class DirectUploadTest extends TestCase
{
    use RefreshDatabase;

    public function test_prepare_returns_presigned_urls(): void
    {
        $user = User::factory()->create();
        $this->actingAs($user);

        $response = $this->postJson('/api/upload/prepare', [
            'items' => [...],
        ]);

        $response->assertOk()
            ->assertJsonStructure([
                'projectId',
                'uploadTargets' => [
                    '*' => ['itemId', 'presignedUrl', 'filePath', 'expiresAt'],
                ],
            ]);
    }

    public function test_complete_validates_file_path(): void
    {
        $user = User::factory()->create();
        $this->actingAs($user);

        // 不正なパス形式
        $response = $this->postJson('/api/upload/complete', [
            'uploads' => [
                ['itemId' => 1, 'filePath' => 'invalid/path.png'],
            ],
        ]);

        $response->assertStatus(400);
    }

    public function test_complete_rejects_mismatched_ids(): void
    {
        $user = User::factory()->create();
        $project = Project::factory()->for($user)->create();
        $item = UploadItem::factory()->for($project)->create();

        $this->actingAs($user);

        // パス内のitem_idとリクエストのitem_idが不一致
        $wrongItemId = $item->id + 999;
        $response = $this->postJson('/api/upload/complete', [
            'uploads' => [
                [
                    'itemId' => $item->id,
                    'filePath' => "uploads/{$project->id}/image_{$wrongItemId}.png",
                ],
            ],
        ]);

        $response->assertStatus(400);
    }
}
```

## まとめ

### Before / After

| 項目 | Before | After |
|------|--------|-------|
| 画像転送 | Base64でAPI経由 | S3直接アップロード |
| アップロード処理 | 同期（直列） | 非同期（並列可能） |
| APIレスポンス | 画像サイズに依存 | 高速（数十ms） |
| サーバー負荷 | 高（デコード・転送） | 低（URL発行のみ） |

### 実装のポイント

1. **Presigned URL**でフロントエンドから直接S3にアップロード
2. **完了通知API**でDB更新と後続処理をトリガー
3. **ファイルパス検証**で改ざんを防止
4. **所有者チェック**でアクセス制御

### 参考リンク

- [Download and upload objects with presigned URLs - Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [Presigned URL best practices - AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/presigned-url-best-practices/overview.html)
- [Securing Amazon S3 presigned URLs for serverless applications - AWS Compute Blog](https://aws.amazon.com/blogs/compute/securing-amazon-s3-presigned-urls-for-serverless-applications/)
- [AWS SDK for PHP - Presigned URLs](https://docs.aws.amazon.com/sdk-for-php/v3/developer-guide/s3-presigned-url.html)
- [Laravel File Storage](https://laravel.com/docs/filesystem)
