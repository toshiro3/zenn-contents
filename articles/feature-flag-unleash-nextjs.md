---
title: "Next.js + Unleash でFeature Flagを実装する - SSR/CSR対応の実践ガイド"
emoji: "🚩"
type: "tech"
topics: ["nextjs", "unleash", "featureflag", "typescript", "react"]
published: true
---

## はじめに

Feature Flag管理ツール「**Unleash**」とNext.jsを組み合わせて、Feature Flagの実装を検証してみました。

本記事では、公式SDK（`@unleash/nextjs`）を使って、SSR/CSR両対応のFeature Flag実装を実際に手を動かしながら確認した結果をまとめます。

### 検証環境

| 項目 | バージョン |
|------|-----------|
| Node.js | 22.22.0 |
| Next.js | 16.1.6 |
| @unleash/nextjs | 最新版 |
| Unleash | Docker版（最新） |

### リポジトリ

検証用のソースコードはこちら：
https://github.com/toshiro3/unleash-nextjs-demo

### 検証項目

今回は以下の4つの機能を検証します。

1. **基本的なON/OFF（SSR）** - サーバーサイドでのフラグ取得
2. **基本的なON/OFF（CSR）** - クライアントサイドでのリアルタイム取得
3. **ユーザーターゲティング** - 特定ユーザーにだけ機能を公開
4. **段階的ロールアウト** - 一定割合のユーザーに機能を公開

## 環境構築

### ディレクトリ構成

検証用のリポジトリは以下のモノレポ構成で作成します。

```
unleash-nextjs-demo/
├── docker-compose.yml    # Unleashサーバー
├── nextjs/               # Next.jsアプリ
│   ├── app/
│   ├── package.json
│   └── .env.local
└── README.md
```

### Unleashサーバーの起動

まず、Unleashサーバーをdocker composeで起動します。

```yaml:docker-compose.yml
services:
  unleash:
    image: unleashorg/unleash-server:latest
    ports:
      - "4242:4242"
    environment:
      DATABASE_URL: postgres://postgres:unleash@db:5432/unleash
      DATABASE_SSL: "false"
      INIT_CLIENT_API_TOKENS: "default:development.unleash-insecure-api-token"
      INIT_FRONTEND_API_TOKENS: "default:development.unleash-insecure-frontend-token"
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: unleash
      POSTGRES_DB: unleash
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 2s
      timeout: 1s
      retries: 5
```

```bash
docker compose up -d
```

起動後、http://localhost:4242 にアクセスし、以下の認証情報でログインします。

- Username: `admin`
- Password: `unleash4all`

### Next.jsプロジェクトの作成

```bash
npx create-next-app@latest nextjs --typescript --tailwind --app --no-src-dir --no-eslint --import-alias "@/*"
cd nextjs
```

Node.jsのバージョンを固定する場合（Volta使用時）:

```bash
volta pin node@22
```

### Unleash SDKのインストール

```bash
npm install @unleash/nextjs
```

### 環境変数の設定

`.env.local`ファイルを作成し、Unleashへの接続情報を設定します。

```bash:.env.local
# Server-side (SSR用)
UNLEASH_SERVER_API_URL=http://localhost:4242/api
UNLEASH_SERVER_API_TOKEN=default:development.unleash-insecure-api-token

# Client-side (CSR用 - Frontend API)
NEXT_PUBLIC_UNLEASH_FRONTEND_API_URL=http://localhost:4242/api/frontend
NEXT_PUBLIC_UNLEASH_FRONTEND_API_TOKEN=default:development.unleash-insecure-frontend-token
```

ポイントは、SSR用とCSR用で**異なるAPIエンドポイントとトークン**を使用することです。

| 用途 | エンドポイント | トークン種別 |
|------|---------------|-------------|
| SSR | `/api` | Server API Token |
| CSR | `/api/frontend` | Frontend API Token |

これで環境構築は完了です。Next.jsを起動して動作確認しましょう。

```bash
npm run dev
```

## 基本的なON/OFF（SSR）

まずは、サーバーサイドレンダリング（SSR）でフラグを取得する方法を検証します。

### Unleashでフラグを作成

1. Unleash管理画面で「Feature flags」→「New feature flag」
2. Name: `new-feature` で作成
3. development環境で「Enable」に切り替え

![](/images/feature-flag-unleash-nextjs/new-feature-flag.png)

### SSRでフラグを取得

App Routerでは、Server Componentで直接フラグを取得できます。

```tsx:app/page.tsx
import { evaluateFlags, flagsClient, getDefinitions } from "@unleash/nextjs";

export default async function Home() {
  const definitions = await getDefinitions();
  const { toggles } = evaluateFlags(definitions);
  const flags = flagsClient(toggles);

  const isNewFeatureEnabled = flags.isEnabled("new-feature");

  return (
    <main className="min-h-screen p-8">
      <h1 className="text-2xl font-bold mb-4">Unleash + Next.js SSR検証</h1>
      
      <div className="p-4 border rounded">
        <p className="mb-2">
          <span className="font-semibold">new-feature:</span>{" "}
          <span className={isNewFeatureEnabled ? "text-green-500" : "text-red-500"}>
            {isNewFeatureEnabled ? "ON" : "OFF"}
          </span>
        </p>
      </div>

      {isNewFeatureEnabled && (
        <div className="mt-4 p-4 bg-green-100 text-green-800 rounded">
          🎉 新機能が有効です！
        </div>
      )}
    </main>
  );
}
```

### コードの解説

1. **`getDefinitions()`**: Unleashサーバーからフラグの定義情報を取得します
2. **`evaluateFlags(definitions)`**: 取得した定義を評価してトグルの状態を計算します
3. **`flagsClient(toggles)`**: トグル情報からフラグクライアントを生成します
4. **`flags.isEnabled("new-feature")`**: 指定したフラグがONかどうかを判定します

:::message
`getDefinitions()` は内部で `process.env.UNLEASH_SERVER_API_URL` と `process.env.UNLEASH_SERVER_API_TOKEN` を自動的に参照します。そのため、環境変数名は正確にこの通りである必要があります。
:::

### 動作確認

http://localhost:3000 にアクセスすると、フラグの状態に応じて表示が切り替わります。

- フラグON時: `new-feature: ON` と緑のボックスが表示
- フラグOFF時: `new-feature: OFF` のみ表示

Unleash管理画面でフラグをON/OFFに切り替え、ページをリロードすると反映されることを確認できます。

:::message
SSRの場合、フラグの変更を反映するには**ページのリロードが必要**です。これは、フラグの評価がサーバーサイドで行われるためです。
:::

## クライアントサイド（CSR）での利用

次に、クライアントサイドでフラグをリアルタイムに取得する方法を検証します。CSRでは、ポーリングによってフラグの変更がリロードなしで反映されます。

### FlagProviderの設定

CSRでフラグを使用するには、アプリケーション全体を`FlagProvider`でラップします。

App Routerでは `layout.tsx` はデフォルトでServer Componentです。`FlagProvider` は内部でReact Contextを使用しているため、Client Componentとして分離します。

まず、Providerコンポーネントを作成します。

```tsx:app/providers.tsx
"use client";

import { FlagProvider } from "@unleash/nextjs/client";

export function Providers({ children }: { children: React.ReactNode }) {
  return <FlagProvider>{children}</FlagProvider>;
}
```

次に、`layout.tsx` でこのProviderを使用します。

```tsx:app/layout.tsx
import type { Metadata } from "next";
import { Geist, Geist_Mono } from "next/font/google";
import "./globals.css";
import { Providers } from "./providers";

const geistSans = Geist({
  variable: "--font-geist-sans",
  subsets: ["latin"],
});

const geistMono = Geist_Mono({
  variable: "--font-geist-mono",
  subsets: ["latin"],
});

export const metadata: Metadata = {
  title: "Unleash + Next.js Demo",
  description: "Feature flag demo with Unleash",
};

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="ja">
      <body
        className={`${geistSans.variable} ${geistMono.variable} antialiased`}
      >
        <Providers>
          {children}
        </Providers>
      </body>
    </html>
  );
}
```

:::message alert
`FlagProvider`は`@unleash/nextjs/client`からインポートする必要があります。`@unleash/nextjs`からのインポートではないので注意してください。
:::

### CSR用のページを作成

Client Componentとして、`useFlag`フックを使ってフラグを取得します。

```tsx:app/csr/page.tsx
"use client";

import { useFlag, useFlagsStatus } from "@unleash/nextjs/client";

export default function CSRPage() {
  const { flagsReady } = useFlagsStatus();
  const isNewFeatureEnabled = useFlag("new-feature");

  if (!flagsReady) {
    return (
      <main className="min-h-screen p-8">
        <p>Loading flags...</p>
      </main>
    );
  }

  return (
    <main className="min-h-screen p-8">
      <h1 className="text-2xl font-bold mb-4">Unleash + Next.js CSR検証</h1>

      <div className="p-4 border rounded">
        <p className="mb-2">
          <span className="font-semibold">new-feature:</span>{" "}
          <span className={isNewFeatureEnabled ? "text-green-500" : "text-red-500"}>
            {isNewFeatureEnabled ? "ON" : "OFF"}
          </span>
        </p>
      </div>

      {isNewFeatureEnabled && (
        <div className="mt-4 p-4 bg-green-100 text-green-800 rounded">
          🎉 新機能が有効です！（CSR）
        </div>
      )}

      <div className="mt-8 text-sm text-gray-500">
        <p>※ CSRではページリロードなしでフラグの変更が反映されます（ポーリング間隔による）</p>
      </div>
    </main>
  );
}
```

### コードの解説

1. **`"use client"`**: Client Componentであることを宣言
2. **`useFlagsStatus()`**: フラグの読み込み状態を取得。`flagsReady`がtrueになるまでローディング表示
3. **`useFlag("new-feature")`**: 指定したフラグの状態をリアクティブに取得

### 動作確認

http://localhost:3000/csr にアクセスし、Unleash管理画面でフラグを切り替えると、**ページをリロードせずに**表示が切り替わることを確認できます（デフォルトのポーリング間隔は約15秒）。

## ユーザーターゲティング

特定のユーザーにだけ機能を公開する「ユーザーターゲティング」を検証します。ベータテストや段階的なリリースで非常に有用な機能です。

### Unleashでターゲティング用フラグを作成

1. 「New feature flag」で `beta-feature` を作成
2. development環境の「Add strategy」をクリック
3. 「Standard」ストラテジーを選択
4. 「Targeting」タブで以下を設定:
   - Context field: `userId`
   - Operator: `is one of`
   - Values: `user-123`
5. 保存してdevelopment環境をEnableに

![](/images/feature-flag-unleash-nextjs/targeting-constraint.png)

この設定により、`userId`が`user-123`のユーザーにのみフラグが有効になります。

### ターゲティング検証ページを作成

```tsx:app/targeting/page.tsx
import { evaluateFlags, flagsClient, getDefinitions } from "@unleash/nextjs";

export default async function TargetingPage() {
  const definitions = await getDefinitions();

  // ユーザーAのコンテキスト（ターゲット対象）
  const userAContext = { userId: "user-123" };
  const userAEvaluated = evaluateFlags(definitions, userAContext);
  const userAFlags = flagsClient(userAEvaluated.toggles);

  // ユーザーBのコンテキスト（ターゲット対象外）
  const userBContext = { userId: "user-456" };
  const userBEvaluated = evaluateFlags(definitions, userBContext);
  const userBFlags = flagsClient(userBEvaluated.toggles);

  return (
    <main className="min-h-screen p-8">
      <h1 className="text-2xl font-bold mb-4">ユーザーターゲティング検証</h1>

      <div className="space-y-4">
        <div className="p-4 border rounded">
          <h2 className="font-semibold mb-2">User A (user-123) - ターゲット対象</h2>
          <p>
            beta-feature:{" "}
            <span className={userAFlags.isEnabled("beta-feature") ? "text-green-500" : "text-red-500"}>
              {userAFlags.isEnabled("beta-feature") ? "ON" : "OFF"}
            </span>
          </p>
        </div>

        <div className="p-4 border rounded">
          <h2 className="font-semibold mb-2">User B (user-456) - ターゲット対象外</h2>
          <p>
            beta-feature:{" "}
            <span className={userBFlags.isEnabled("beta-feature") ? "text-green-500" : "text-red-500"}>
              {userBFlags.isEnabled("beta-feature") ? "ON" : "OFF"}
            </span>
          </p>
        </div>
      </div>
    </main>
  );
}
```

### コードの解説

ポイントは、`evaluateFlags()`の第2引数に**コンテキスト**を渡すことです。

```typescript
const userAContext = { userId: "user-123" };
const userAEvaluated = evaluateFlags(definitions, userAContext);
```

コンテキストには以下のような情報を含めることができます：

| フィールド | 説明 | 例 |
|-----------|------|-----|
| userId | ユーザーID | "user-123" |
| sessionId | セッションID | "session-abc" |
| remoteAddress | IPアドレス | "192.168.1.1" |
| properties | カスタムプロパティ | { plan: "premium" } |

### 動作確認

http://localhost:3000/targeting にアクセスすると:

- User A (user-123): `beta-feature: ON`
- User B (user-456): `beta-feature: OFF`

同じフラグでも、ユーザーによって結果が異なることを確認できます。

### 実際のアプリケーションでの使用例

実際のアプリケーションでは、認証情報からuserIdを取得してコンテキストに設定します。

```tsx
import { auth } from "@/lib/auth";

export default async function Page() {
  const session = await auth();
  const definitions = await getDefinitions();
  
  const context = { userId: session?.user?.id };
  const evaluated = evaluateFlags(definitions, context);
  const flags = flagsClient(evaluated.toggles);
  
  // ...
}
```

## 段階的ロールアウト

最後に、ユーザーの一定割合にだけ機能を公開する「段階的ロールアウト」を検証します。新機能のリスクを軽減しながら徐々に展開する際に使用します。

### Unleashでロールアウト用フラグを作成

1. 「New feature flag」で `gradual-rollout` を作成
2. development環境の「Add strategy」をクリック
3. 「Gradual rollout」を選択
4. 以下を設定:
   - Rollout: `50%`
   - Stickiness: `default`
   - groupId: `gradual-rollout`（自動入力）
5. 保存してdevelopment環境をEnableに

![](/images/feature-flag-unleash-nextjs/gradual-rollout.png)

### 段階的ロールアウト検証ページを作成

```tsx:app/rollout/page.tsx
import { evaluateFlags, flagsClient, getDefinitions } from "@unleash/nextjs";

export default async function RolloutPage() {
  const definitions = await getDefinitions();

  // 10人のユーザーでフラグを評価
  const users = Array.from({ length: 10 }, (_, i) => `user-${i + 1}`);

  const results = users.map((userId) => {
    const context = { userId };
    const evaluated = evaluateFlags(definitions, context);
    const flags = flagsClient(evaluated.toggles);
    return {
      userId,
      enabled: flags.isEnabled("gradual-rollout"),
    };
  });

  const enabledCount = results.filter((r) => r.enabled).length;

  return (
    <main className="min-h-screen p-8">
      <h1 className="text-2xl font-bold mb-4">段階的ロールアウト検証（50%）</h1>

      <div className="mb-4 p-4 bg-gray-800 rounded">
        <p>
          有効なユーザー: {enabledCount} / {users.length} ({(enabledCount / users.length) * 100}%)
        </p>
      </div>

      <div className="grid grid-cols-2 gap-4">
        {results.map(({ userId, enabled }) => (
          <div key={userId} className="p-4 border rounded">
            <p>
              <span className="font-semibold">{userId}:</span>{" "}
              <span className={enabled ? "text-green-500" : "text-red-500"}>
                {enabled ? "ON" : "OFF"}
              </span>
            </p>
          </div>
        ))}
      </div>

      <div className="mt-8 text-sm text-gray-500">
        <p>※ 同じuserIdは常に同じ結果になります（Stickiness）</p>
        <p>※ 50%設定でも、サンプル数が少ないと偏りが出ることがあります</p>
      </div>
    </main>
  );
}
```

### 動作確認

http://localhost:3000/rollout にアクセスすると、10人中およそ半分のユーザーがONになっていることを確認できます。

重要なポイントは**Stickiness（粘着性）**です：

- 同じuserIdは、何度評価しても常に同じ結果になる
- ページをリロードしても結果は変わらない
- これにより、ユーザー体験の一貫性が保たれる

### ロールアウト率の調整

段階的にリリースを進める場合、Unleash管理画面でRollout率を調整します：

1. 10% → 初期リリース、問題がないか監視
2. 30% → 問題なければ拡大
3. 50% → さらに拡大
4. 100% → 全ユーザーにリリース

問題が発生した場合は、即座に0%に戻すことでロールバックできます。

## SSR vs CSR の使い分け

### SSRを使うべきケース

| ケース | 理由 |
|-------|------|
| 初期表示の最適化 | フラグの結果が最初からHTMLに含まれる |
| SEO対応 | クローラーがフラグ適用後のコンテンツを見れる |
| ユーザーコンテキストがサーバーで決まる | 認証情報などをサーバーで取得する場合 |

```tsx
// SSRでの典型的なパターン
export default async function Page() {
  const session = await auth();
  const definitions = await getDefinitions();
  const evaluated = evaluateFlags(definitions, { userId: session?.user?.id });
  const flags = flagsClient(evaluated.toggles);
  
  return flags.isEnabled("feature") ? <NewFeature /> : <OldFeature />;
}
```

### CSRを使うべきケース

| ケース | 理由 |
|-------|------|
| リアルタイム更新 | リロードなしでフラグ変更を反映したい |
| A/Bテスト | ユーザーセッション中にフラグを変更する可能性がある |
| クライアント固有のコンテキスト | ブラウザ情報などをコンテキストに含める場合 |

```tsx
"use client";

// CSRでの典型的なパターン
export default function Component() {
  const { flagsReady } = useFlagsStatus();
  const isEnabled = useFlag("feature");
  
  if (!flagsReady) return <Loading />;
  
  return isEnabled ? <NewFeature /> : <OldFeature />;
}
```

### ハイブリッドアプローチ

多くの場合、SSRで初期値を取得し、CSRで更新を受け取るハイブリッドアプローチが最適です。`@unleash/nextjs`はこのパターンもサポートしています。

## トラブルシューティング

### 開発モードで警告が表示される

`npm run dev`（Turbopack）実行時に以下の警告が表示されることがあります。

```
Using fallback Unleash API URL (http://localhost:4242/api). Provide a URL or set UNLEASH_SERVER_API_URL environment variable.
Using fallback default token. Pass token or set UNLEASH_SERVER_API_TOKEN environment variable.
```

これはTurbopackがモジュールを高速に評価する際、環境変数のマッピングより一瞬早くSDKの初期化が走るために発生するTurbopack開発モード特有の動作です。

フォールバック値がローカル開発用の設定と同じため、**機能には影響ありません**。ローカルで `npm run build && npm run start` を実行して警告が出ないことを確認できていれば、本番環境での動作に問題はありません。

### 環境変数が読み込まれない

`.env.local`を作成・変更した後は、Next.jsの開発サーバーを**再起動**する必要があります。

```bash
# Ctrl+C で停止してから
npm run dev
```

### フラグがfalseになる

以下を確認してください：

1. Unleash管理画面で該当環境（development）がEnableになっているか
2. ストラテジーが追加されているか
3. APIトークンが正しいか
4. 環境変数名が正しいか（`UNLEASH_SERVER_API_TOKEN`など）

### CSRでフラグが取得できない

1. `FlagProvider`でアプリをラップしているか確認
2. Frontend API Token（`NEXT_PUBLIC_`で始まる）が設定されているか確認
3. ブラウザの開発者ツールでネットワークエラーを確認

## まとめ

本記事では、Next.js（App Router）とUnleashを連携してFeature Flagを実装する方法を検証しました。

### 検証結果

| 項目 | 結果 |
|------|------|
| 基本ON/OFF（SSR） | ✅ 正常動作 |
| 基本ON/OFF（CSR） | ✅ 正常動作、リアルタイム更新も確認 |
| ユーザーターゲティング | ✅ Constraintsで実現可能 |
| 段階的ロールアウト | ✅ Stickinessも正常動作 |

### @unleash/nextjs SDKの特徴

- **App Router対応**: Server ComponentとClient Componentの両方で利用可能
- **シンプルなAPI**: `getDefinitions()` → `evaluateFlags()` → `flagsClient()` の3ステップ
- **コンテキスト対応**: ユーザーIDなどを渡してターゲティング可能
- **ポーリング対応**: CSRではリアルタイムにフラグ変更を検知

### 今後の展望

今回の検証でNext.jsとUnleashの基本的な連携が確認できました。時間を見つけて、以下の機能も検証してみようと思います。

- **Unleash Edge/Proxyの導入**: 大規模環境でのパフォーマンスとセキュリティ向上
- **メトリクス収集**: フラグの評価結果をUnleashに送信して効果測定
- **型安全性の向上**: SDKのCLIを使った型生成

## 参考資料

- [Unleash公式ドキュメント](https://docs.getunleash.io/)
- [Unleash Next.js SDK](https://docs.getunleash.io/sdks/next-js)
- [GitHub: @unleash/nextjs](https://github.com/Unleash/unleash-client-nextjs)
