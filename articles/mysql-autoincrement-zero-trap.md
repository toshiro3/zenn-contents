---
title: "MySQLでid=0をINSERTすると別のIDが入る罠"
emoji: "🔢"
type: "tech"
topics: ["mysql", "laravel", "php", "テスト"]
published: false
---

## はじめに

テストが不安定になる原因を調査していたところ、MySQLの意外な挙動に遭遇しました。`AUTO_INCREMENT`カラムに`0`を指定してINSERTすると、`0`ではなく**次の連番が割り当てられる**という仕様です。

この挙動を知らないと、特にテストコードで予期しない問題を引き起こす可能性がありそうです。

## 現象：id=0でINSERTしたら別のIDが入った

まず、実際の挙動を確認してみましょう。

```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255)
);

-- id=0を指定してINSERT
INSERT INTO users (id, name) VALUES (0, 'Alice');

-- 結果を確認
SELECT * FROM users;
```

期待する結果：
| id | name |
|----|------|
| 0 | Alice |

実際の結果：
| id | name |
|----|------|
| 1 | Alice |

**`id=0`を指定したのに、`id=1`が入っています。**

## MySQLの仕様

これはMySQLの仕様です。[公式ドキュメント](https://dev.mysql.com/doc/refman/8.0/en/example-auto-increment.html)には以下のように記載されています。

> When you insert any other value into an AUTO_INCREMENT column, the column is set to that value and the sequence is reset so that the next automatically generated value follows sequentially from the inserted value.
>
> **Inserting NULL or 0 into the AUTO_INCREMENT column causes the next sequence value to be inserted.**

つまり、`AUTO_INCREMENT`カラムに対して：

| 指定値 | 動作 |
|--------|------|
| `NULL` | 次の連番を割り当て |
| `0` | 次の連番を割り当て（NULLと同じ） |
| 正の整数 | 指定した値をそのまま使用 |

`0`と`NULL`は同じ扱いになるのです。

## 遭遇しそうなケース

### Factoryでfaker->randomNumber()を使っている場合

外部キー制約がない場合、Factoryで「とりあえず数値が入ればいい」という実装をしたとします。

```php
class PostFactory extends Factory
{
    public function definition(): array
    {
        return [
            'title' => $this->faker->sentence,
            'category_id' => $this->faker->randomNumber(),
        ];
    }
}
```

一見問題なさそうですが、`randomNumber()`は**0を返す可能性があります**。

```php
// randomNumber()の戻り値の例
$this->faker->randomNumber();  // 0, 1, 42, 12345, ...
```

0が返された場合、前述のMySQLの仕様により：
- `category_id = 0`でレコードが作成される
- しかし`id = 0`の`Category`は存在しない（`AUTO_INCREMENT`は1から始まる）
- リレーションが壊れた状態になる

外部キー制約がないため、このデータは静かに作成されてしまいます。

### 明示的に0を設定している場合

Factoryで外部キーのデフォルト値が明示的に`0`になっているケースです。

```php
class PostFactory extends Factory
{
    public function definition(): array
    {
        return [
            'title' => $this->faker->sentence,
            'category_id' => 0,  // 本来は有効なIDが必要
        ];
    }
}
```

この状態で以下のようなテストを書くと：

```php
public function test_post_belongs_to_category()
{
    // カテゴリを作成（id=1が割り当てられる）
    $category = Category::factory()->create();

    // 投稿を作成（category_id=0でINSERTされる）
    $post = Post::factory()->create();

    // リレーションを確認
    $this->assertEquals($category->id, $post->category->id);  // 失敗！
}
```

`Post`の`category_id`は`0`のままですが、`Category`の`id`は`1`になっているため、リレーションが壊れます。

### テスト実行順序による問題

問題が顕在化しにくいパターンもあります。

```php
// テストAが先に実行される
public function test_a()
{
    $category = Category::factory()->create();  // id=1
    $post = Post::factory()->create([
        'category_id' => 0,  // 0が保存される
    ]);
    // category_id=0 なので、categoryとの紐付けはない
    // でもテスト自体は別の観点なのでパスする
}

// テストBが後に実行される
public function test_b()
{
    $post = Post::factory()->create([
        'category_id' => 0,  // 0が保存される
    ]);
    $category = Category::factory()->create();  // id=1

    // たまたまcategory_id=0で、categoryはid=1
    // リレーションは壊れているが、このテストでは検証していない
}
```

テストの実行順序やデータベースの状態によって、問題が発生したりしなかったりする「不安定なテスト」の原因になり得ます。

:::message alert
特に厄介なのは、外部キー制約がない場合です。制約があれば`0`は即座にエラーになりますが、制約がないと静かに壊れたデータが作成されます。
:::

## 対策方法

### 1. Factoryで適切な関連を設定する（推奨）

```php
class PostFactory extends Factory
{
    public function definition(): array
    {
        return [
            'title' => $this->faker->sentence,
            // 関連するレコードを自動作成
            'category_id' => Category::factory(),
        ];
    }
}
```

### 2. Factoryでnullableにする

関連が必須でない場合は、`null`を明示的に設定します。

```php
class PostFactory extends Factory
{
    public function definition(): array
    {
        return [
            'title' => $this->faker->sentence,
            'category_id' => null,  // 0ではなくnull
        ];
    }
}
```

### 3. 外部キー制約を追加する

データベースレベルで不正な値を防ぎます。

```php
Schema::table('posts', function (Blueprint $table) {
    $table->foreign('category_id')
          ->references('id')
          ->on('categories');
});
```

### 4. NO_AUTO_VALUE_ON_ZERO モード

MySQLには`NO_AUTO_VALUE_ON_ZERO`というSQLモードがあり、これを有効にすると`0`を`0`としてINSERTできます。

```sql
SET sql_mode = 'NO_AUTO_VALUE_ON_ZERO';
INSERT INTO users (id, name) VALUES (0, 'Alice');
-- id=0で保存される
```

ただし、この設定は影響範囲が大きいため、既存システムへの導入は慎重に検討してください。

:::message
`NO_AUTO_VALUE_ON_ZERO`は、他のシステムからデータを移行する際など、`0`をIDとして使用する必要がある場合に有用です。通常の開発では、`0`をIDとして使わない設計にする方が安全です。
:::

## 他のRDBMSとの比較

| RDBMS | id=0でINSERTした場合 |
|-------|---------------------|
| MySQL | 次の連番が割り当てられる |
| PostgreSQL | 0がそのまま保存される |
| SQLite | 0がそのまま保存される |

MySQLだけが特殊な挙動をします。PostgreSQLやSQLiteから移行する際は注意が必要です。

```sql
-- PostgreSQLの場合
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255)
);

INSERT INTO users (id, name) VALUES (0, 'Alice');
SELECT * FROM users;
-- id=0, name='Alice' がそのまま保存される
```

```sql
-- SQLiteの場合
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    name TEXT
);

INSERT INTO users (id, name) VALUES (0, 'Alice');
SELECT * FROM users;
-- id=0, name='Alice' がそのまま保存される
```

:::message
SQLiteでは`INTEGER PRIMARY KEY`と指定すると自動採番になります。`AUTOINCREMENT`を明示的に付けることもできますが、どちらも`id=0`はそのまま保存されます。
:::

## まとめ

- MySQLの`AUTO_INCREMENT`カラムに`0`を指定すると、`NULL`と同様に次の連番が割り当てられる
- これはMySQLの仕様であり、PostgreSQLやSQLiteとは異なる挙動
- Factoryで外部キーに`0`を設定してしまうと、テストが不安定になる原因になり得る
- 対策としては、Factoryで適切な関連を設定するか、外部キー制約を追加する

一見すると些細な仕様ですが、知らないとデバッグに時間を取られることがあります。特にテストの不安定さに悩んでいる場合は、この挙動を疑ってみてください。

## 参考リンク

- [MySQL 8.0 Reference Manual - Using AUTO_INCREMENT](https://dev.mysql.com/doc/refman/8.0/en/example-auto-increment.html)
- [MySQL 8.0 Reference Manual - Server SQL Modes](https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html#sqlmode_no_auto_value_on_zero)
