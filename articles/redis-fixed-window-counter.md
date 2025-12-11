---
title: "LaravelでRedisの固定ウィンドウカウンターを実装したら、4回書き直すことになった話"
emoji: "🔢"
type: "tech"
topics: ["Laravel", "Redis", "PHP", "レートリミット"]
published: true
---

## はじめに

「直近1分間のリクエスト数をカウントしたい」

一見シンプルな要件ですが、実際に実装してみると意外な落とし穴がありました。本記事では、LaravelとRedisを使った固定ウィンドウ方式カウンターの実装で踏んだ失敗と、最終的にたどり着いた解決策を紹介します。

### この記事で扱うこと

- 固定ウィンドウ方式カウンターの基本的な考え方
- `Cache::put()` や `Cache::add()` の挙動の違い
- 競合状態（Race Condition）が起きるパターン
- Luaスクリプトによるアトミックな実装

### 想定読者

- LaravelでRedisを使っている方
- キャッシュ操作の競合状態に悩んだことがある方
- レートリミットやトラフィック監視を自前で実装したい方

## 背景

とあるAPIのトラフィック状況をリアルタイムに把握するため、直近1分間のリクエスト数をカウントする機能を実装することになりました。要件をまとめると以下の通りです。

- リクエストごとにカウンターをインクリメント
- 1分経過したらカウンターをリセット
- カウント値に応じて負荷状況を判定し、必要に応じてユーザーへフィードバック

いわゆる**固定ウィンドウ方式**のカウンターです。

## 実装1: `Cache::put()` による素朴な実装

最初に書いたコードはこちらです。

```php
public static function increment(): void
{
    $count = (int) Cache::get(self::CACHE_KEY, 0);
    Cache::put(self::CACHE_KEY, $count + 1, 60);
}
```

動作確認では問題なさそうに見えました。が、これには致命的な問題があります。

### 問題: TTLが毎回リセットされる

`Cache::put()` は値をセットするたびにTTLを上書きします。つまり、リクエストが来るたびに有効期限が60秒後に延長されてしまいます。

```
T=0  リクエスト → カウント=1, TTL=60秒後
T=30 リクエスト → カウント=2, TTL=60秒後（リセット！）
T=60 リクエスト → カウント=3, TTL=60秒後（まだ消えない）
```

継続的にリクエストがある限りキーは消えません。「直近1分間」のカウントではなく、**最初のリクエストからの累積**になってしまいました。

## 実装2: `Cache::add()` + `Cache::increment()`

TTLをリセットしないよう、`Cache::add()` を使う方法に変更しました。

```php
public static function increment(): void
{
    Cache::add(self::CACHE_KEY, 0, 60);
    Cache::increment(self::CACHE_KEY);
}
```

`Cache::add()` はキーが存在しない場合のみ値とTTLをセットします。既存キーがあれば何もしません。これで固定ウィンドウが実現できる……と思いました。

### 問題: 競合状態（Race Condition）

2つの操作が分離しているため、以下のような競合が発生する可能性があります。

```
【前提】キーが期限切れで消えた直後

T1 リクエストA: Cache::add() を実行しようとする
T2 リクエストB: Cache::increment() を先に実行 → キー=1（TTLなし！）
T3 リクエストA: Cache::add() 失敗（キーが存在するため）
T4 リクエストA: Cache::increment() → キー=2（TTLなし）
```

`Cache::increment()` は存在しないキーに対して実行すると、**TTLなしでキーを作成**します。このタイミングで別リクエストの `add()` が失敗すると、TTLが設定されないままカウンターが永続化されてしまいます。

実際にこの問題が起きるかは確率的ですが、リクエストが集中するタイミングほど発生しやすくなりそうです。

## 実装3: MULTI/EXECトランザクション

競合を防ぐため、Redisのトランザクション機能を使ってみました。

```php
public static function increment(): void
{
    $redis = Cache::getStore()->connection();
    $key = Cache::getPrefix() . self::CACHE_KEY;

    $redis->watch($key);
    $exists = $redis->exists($key);

    $redis->multi();
    $redis->incr($key);
    if (!$exists) {
        $redis->expire($key, 60);
    }
    $redis->exec();
}
```

`WATCH` で楽観的ロックをかけ、`MULTI/EXEC` でアトミックに実行する作戦です。

### 問題: 高負荷時に失敗する

`WATCH` は監視対象のキーが変更されると、`EXEC` 時にトランザクション全体が失敗します（`null` が返る）。

つまり、**カウンターが最も必要な高負荷時に失敗しやすい**という本末転倒な状態になります。リトライロジックを追加すればなんとかなりますが、コードが複雑化する上にネットワークラウンドトリップも多くなります（WATCH → EXISTS → MULTI → INCR → EXPIRE → EXEC）。

## 実装4: Luaスクリプト（最終形）

最終的にたどり着いたのが、Luaスクリプトによる実装です。

```php
public static function increment(): void
{
    $store = Cache::getStore();

    if ($store instanceof RedisStore) {
        $redis = $store->connection();
        $key = Cache::getPrefix() . self::CACHE_KEY;

        $script = <<<'LUA'
            local current = redis.call('INCR', KEYS[1])
            if current == 1 then
                redis.call('EXPIRE', KEYS[1], ARGV[1])
            end
            return current
        LUA;

        $redis->eval($script, 1, $key, self::WINDOW_SIZE);
        return;
    }

    // Redis以外（テスト環境のarrayドライバ等）の場合はフォールバック
    Cache::add(self::CACHE_KEY, 0, self::WINDOW_SIZE);
    Cache::increment(self::CACHE_KEY);
}
```

### Luaスクリプトの解説

```lua
local current = redis.call('INCR', KEYS[1])  -- キーをインクリメント
if current == 1 then                          -- 結果が1 = 新しいウィンドウの開始
    redis.call('EXPIRE', KEYS[1], ARGV[1])    -- TTLを設定
end
return current
```

ポイントは `INCR` の戻り値を利用している点です。

- `INCR` は存在しないキーを `0` として扱い、インクリメント後の値を返す
- 戻り値が `1` なら、キーが新規作成されたことを意味する
- 新規作成時のみ `EXPIRE` でTTLを設定

### なぜLuaスクリプトで解決するのか

Redisは**シングルスレッド**でコマンドを処理します。Luaスクリプトは1つのコマンドとして扱われるため、スクリプト実行中に他のコマンドが割り込むことはありません。

これにより、以下のメリットが得られます。

| 項目 | 説明 |
|------|------|
| 完全なアトミック性 | `INCR` と `EXPIRE` が必ずセットで実行される |
| 競合状態なし | 他のクライアントの操作が割り込めない |
| ネットワーク効率 | 1回のラウンドトリップで完結 |
| シンプル | リトライロジック不要 |

### Luaスクリプト利用時の注意点

RedisでLuaスクリプトを使う場合、以下の点に注意が必要です。

- `eval()` の第2引数でキー数を正しく指定する必要がある
- 長時間実行されるスクリプトはブロッキングの原因になるため、短く保つ

今回のスクリプトは2コマンドだけなので大きな問題はないと考えていますが、Luaスクリプトを本番運用するのは初めてのため、見落としている点があるかもしれません。運用経験のある方からのアドバイスがあれば、ぜひコメントで教えていただけると嬉しいです。

:::message
筆者の環境はAWS ElastiCache（[Valkey](https://valkey.io/)互換）です。Valkeyは2024年にRedisからフォークされたOSSで、Luaスクリプトも同様に利用できます。
:::

## 実装の比較

| 実装 | TTLリセット | 競合状態 | 複雑度 | ラウンドトリップ |
|------|:-----------:|:--------:|:------:|:---------------:|
| `Cache::put()` | ❌ あり | - | 低 | 2回 |
| `Cache::add()` + `increment()` | ✅ なし | ❌ あり | 低 | 2回 |
| MULTI/EXEC | ✅ なし | △ リトライ必要 | 高 | 4〜5回 |
| Luaスクリプト | ✅ なし | ✅ なし | 中 | 1回 |

## 補足: 固定ウィンドウ方式の限界

固定ウィンドウ方式には「境界問題」があります。

```
ウィンドウ1: 0:00〜0:59 → 100リクエスト
ウィンドウ2: 1:00〜1:59 → 100リクエスト

0:30〜1:30の1分間で見ると → 最大200リクエスト！
```

ウィンドウ境界をまたぐスパイクを検出できない場合があります。

より精度の高い**スライディングウィンドウ方式**もありますが、Sorted Setを使った実装が必要になり複雑化します。今回は負荷状況の目安を把握できれば十分だったので、固定ウィンドウ方式を採用しました。

## まとめ

- `Cache::put()` はTTLをリセットするので、固定ウィンドウには使えない
- `Cache::add()` + `Cache::increment()` は競合状態が発生しうる
- MULTI/EXECは混雑時に失敗しやすい
- **Luaスクリプトならアトミックかつ効率的に実装できる**

「たかがカウンター」と思って始めた実装でしたが、分散システムにおける競合状態の難しさを改めて実感しました。同じような要件に直面している方の参考になれば幸いです。
