---
title: "Laravel組み込みの同時実行制御"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["PHP", "Laravel"]
published: true
published_at: 2020-12-12
---

これは[OPENLOGI Advent Calendar 2020](https://qiita.com/advent-calendar/2020/openlogi)の13日目の記事です。

## はじめに

Laravelには、Redisを利用して、同じ処理を同時にいくつ実行できるかを制御するためのモジュールがあります。
弊社では、現在laravelの7.xを使っていますが、7.xの情報については、基本的には以下を参照いただければ理解できると思います。
[https://readouble.com/laravel/7.x/ja/queues.html](https://readouble.com/laravel/7.x/ja/queues.html)

基本的にはドキュメントを読めばだいたいのことを理解できるのですが、やや言葉足らずな部分があります。
今回は、それらのモジュールの同時実行制御が、実際にどのように動くのか、検証していきます。
また、この記事ではlaravel 7.xを前提としています。

## 説明対象

まず、どこから利用するかですが、`Illuminate\Support\Facades\Redis`というfacadeが用意されており、そこから利用できます。
また、今回解説する関数については、`Illuminate\Redis\Connections\Connection`に関数の定義があります。

で、利用する関数としては、主に2つあります。
`funnel`と`throttle`です。
一つ一つ解説していきます。

## funnel

funnelは、一度に実行できるジョブの数を制限するというものです。
実際の実装は以下のクラスに定義されています。

- `Illuminate\Redis\Limiters\ConcurrencyLimiter`
- `Illuminate\Redis\Limiters\ConcurrencyLimiterBuilder`

Redisあんまり詳しく無いのですが、redis操作する際にコマンドを打つのを、一連のまとめた処理として、luaスクリプトで実行できるようです。
RDBで言えば、ストアドプロシージャのようなものでしょうか？
php上にコードとして定義されているので、Redis側に登録済みの状態で実行されるのではなく、都度スクリプト自体をRedis上で実行しているっぽいので、厳密には違うでしょう。

具体的にはevalというコマンドを使っています。
ドキュメントは以下を見るとよさそうです。
[https://redis.io/commands/eval](https://redis.io/commands/eval)

funnelで内部的に呼び出しているluaのscriptは以下の2つです。

### lockScript
```lua
for index, value in pairs(redis.call('mget', unpack(KEYS))) do
    if not value then
        redis.call('set', KEYS[index], ARGV[3], "EX", ARGV[2])
        return ARGV[1]..index
    end
end
```

### releaseScript
```lua
if redis.call('get', KEYS[1]) == ARGV[1]
then
    return redis.call('del', KEYS[1])
else
    return 0
end
```

と、書いたものの、Luaは大昔にProgramming in Luaを読んだことがある程度なので、まじでわかりません。
察するにlockScriptについては、特定のキーがなければ、特定の期限つきのレコードを登録し、登録した結果を返す。

releaseScriptについては、特定のキーに値があり、それが特定の値と一致するのであれば、それを消すという機能を持つようです。
詳しくは、コード読んでください。

上記のコードから推察するに、実行開始時にRedisにキーを登録し、それが登録されている状態では、同時に同じ処理が走らないように制御できる。
また実行が終わったら、キーを削除し、処理を実行可能な状態に戻すことができる。というもののようです。

では、それを実際に試してみたいと思います。

### 挙動

funnelはlimit,releaseAfter,blockという関数で、動きを制御することができるのですが、それぞれ見ていきたいと思います。
こんな感じのコードを使っていきます。

```php
<?php
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Contracts\Redis\LimiterTimeoutException;
use Illuminate\Support\Facades\Redis;

class LaravelRedisTest implements ShouldQueue
{
    /** @var int */
    public $tries = 1;

    /** @var string */
    private $name;

    public function __construct(string $name)
    {
        $this->name = $name;
    }

    public function handle()
    {
        $redis = Redis::connection();
        $redis->funnel(self::class)
            ->limit(2)
            ->block(0)
            ->then(function () {
                sleep(3);
                logger("{$this->name} is success");
            }, function (LimiterTimeoutException $e) {
                logger("{$this->name} is failed");
            });
    }
}
```

#### limit
まずはlimitです。これは同時にいくつ実行できるかを指定します。
同時にblockも使っていますが、これはあとで説明します。

以下の処理をした際の期待値を先に書いておきます。
limitを指定することで、指定数までの処理は実行でき、それを超えると失敗することがわかります。

1. 処理A(3秒)を実行 -> 成功
2. 処理B(3秒)を実行 -> 成功
3. 処理C(3秒)を実行 -> 失敗
4. 処理Aを終了(3秒まつ)
5. 処理D(3秒)を実行 -> 成功

実行スクリプト

```php
function run()
{
    dispatch(new LaravelRedisTest('A'));
    dispatch(new LaravelRedisTest('B'));
    dispatch(new LaravelRedisTest('C'));
    sleep(3);
    dispatch(new LaravelRedisTest('D'));
}
```

実行部分

```php
$redis = LaravelRedis::connection();
$redis->funnel(self::class)
    ->limit(2)
    ->block(0)
    ->then(function () {
        sleep(3);
        echo "{$this->name} is success" . PHP_EOL;
    }, function (LimiterTimeoutException $e) {
        echo "{$this->name} is failed" . PHP_EOL;
    });
```

結果

```
[2020-12-12 15:56:19] local.DEBUG: C is failed  
[2020-12-12 15:56:21] local.DEBUG: A is success  
[2020-12-12 15:56:21] local.DEBUG: B is success  
[2020-12-12 15:56:25] local.DEBUG: D is success  
```

#### releaseAfter
releaseAfterに値を指定すると、その時間が立つとロックを解除して、他の処理を実行できるようになるようです。
なので、長い処理が動く際は、気をつけて指定する必要がありそうです。
またdefaultでは60秒なので指定しなければ60秒立つと勝手にロックが解除されます。

手順と期待値です。
releaseAfterを指定することで、実行中であってもロックが解除されて、実行可能になっていることがわかります。

1. 処理A(3秒)を実行
2. 処理B(3秒)を実行
3. 1秒待つ
4. 処理C(3秒)を実行 -> 成功

```php
function run()
{
    dispatch(new LaravelRedisTest('A'));
    dispatch(new LaravelRedisTest('B'));
    sleep(1);
    dispatch(new LaravelRedisTest('C'));
}
```

```php
$redis = LaravelRedis::connection();
$redis->funnel(self::class)
    ->limit(2)
    ->releaseAfter(1)
    ->block(0)
    ->then(function () {
        sleep(3);
        echo "{$this->name} is success" . PHP_EOL;
    }, function (LimiterTimeoutException $e) {
        echo "{$this->name} is failed" . PHP_EOL;
    });
```

```
[2020-12-12 16:10:52] local.DEBUG: A is success  
[2020-12-12 16:10:52] local.DEBUG: B is success  
[2020-12-12 16:10:53] local.DEBUG: C is success  
```

#### block
blockは、ロックが取れない場合に、どれくらい待つかという感じです。
これがdefaultが3秒になっているので、何も指定しなければ、3秒待つ形になります。
limitやrelaseAfterの項でblockに0を指定していたのは、挙動が複雑になるためでした。

手順と期待値です。
blockを指定することで、ロックが解除されるのを待っているのがわかります。

1. 処理A(3秒)を実行
2. 処理B(3秒)を実行 -> 成功
3. 処理C(3秒)を実行 -> 成功

```php
function run()
{
    dispatch(new LaravelRedisTest('A'));
    dispatch(new LaravelRedisTest('B'));
    dispatch(new LaravelRedisTest('C'));
}
```

```php
$redis = LaravelRedis::connection();
$redis->funnel(self::class)
    ->limit(2)
    ->block(3)
    ->then(function () {
        sleep(3);
        echo "{$this->name} is success" . PHP_EOL;
    }, function (LimiterTimeoutException $e) {
        echo "{$this->name} is failed" . PHP_EOL;
    });
```

```
[2020-12-12 16:16:03] local.DEBUG: A is success  
[2020-12-12 16:16:03] local.DEBUG: B is success  
[2020-12-12 16:16:06] local.DEBUG: C is success  
```

## throttle

throttleはfunnelと違い、時間の概念が追加されます。
単位時間に、いくつの処理を同時に実行するか制御することができるようです。

実装は以下のクラスに存在します。

- `Illuminate\Redis\Limiters\DurationLimiter`
- `Illuminate\Redis\Limiters\DurationLimiterBuilder`

### redis script

それではRedisに投げているLuaスクリプトを読んで見ましょう。

```lua
local function reset()
    redis.call('HMSET', KEYS[1], 'start', ARGV[2], 'end', ARGV[2] + ARGV[3], 'count', 1)
    return redis.call('EXPIRE', KEYS[1], ARGV[3] * 2)
end
if redis.call('EXISTS', KEYS[1]) == 0 then
    return {reset(), ARGV[2] + ARGV[3], ARGV[4] - 1}
end
if ARGV[1] >= redis.call('HGET', KEYS[1], 'start') and ARGV[1] <= redis.call('HGET', KEYS[1], 'end') then
    return {
        tonumber(redis.call('HINCRBY', KEYS[1], 'count', 1)) <= tonumber(ARGV[4]),
        redis.call('HGET', KEYS[1], 'end'),
        ARGV[4] - redis.call('HGET', KEYS[1], 'count')
    }
end
return {reset(), ARGV[2] + ARGV[3], ARGV[4] - 1}
```

funnelは2つスクリプトがありましたが、throttleは1つだけです。
ARGVとか、KEYSとか変数が色々あってもうわからんですね。
それらの変数はphpのコードを読めば分かるのですが、それはコードをご自身で読んでください。

簡単に説明すると、上記のreset関数が、有効期限つきで、登録時と削除時と、同時実行数を1として登録しています。
特定のキーが存在するとき、または存在していても有効期限外であれば、reset関数でキーを登録する。
対象のキーが存在し、その有効期限内であれば、新しくキーを登録したり更新はしない。
返却する値は、新しく実行可能か、現在いくつの処理を並行で処理しようとしているか、また現在の有効期限です。

単位時間に実行する数を制限してくれそうな雰囲気がありますね。
ただ、任意の期間に対して同時に実行している数を制御する、という形ではなさそうです。
あくまで、特定のキーに対して実行しているものが無い際に、最初に実行されたタイミングから指定時間は、同時実行数を制限してくれる。
というものの用に感じます。

### throttleの挙動
funnelと違い、指定する部分の違いではなく、時間が立つと自動的に再実行できるという部分から検証していこうと思います。

#### 処理時間が短い場合
まずは、処理時間が3秒と、指定時間に対して短い、あるいは同等の場合の挙動を見ておきます。
これはわかりやすいはず。

以下、手順と期待値です。
3秒間の間に、2つ以上実行されれば、失敗するし、3秒立てば成功するのが確認できます。

1. 処理A(3秒)を実行
2. 処理B(3秒)を実行
3. 処理C(3秒)を実行 -> 失敗
4. 処理Aを終了(3秒まつ)
5. 処理D(3秒)を実行 -> 成功

```php
function run()
{
    dispatch(new LaravelRedisTest('A'));
    dispatch(new LaravelRedisTest('B'));
    dispatch(new LaravelRedisTest('C'));
    sleep(3);
    dispatch(new LaravelRedisTest('D'));
}
```

```php
$redis = LaravelRedis::connection();
$redis->throttle(self::class)
    ->allow(2)
    ->every(3)
    ->block(0)
    ->then(function () {
        sleep(3);
        echo "{$this->name} is success" . PHP_EOL;
    }, function (LimiterTimeoutException $e) {
        echo "{$this->name} is failed" . PHP_EOL;
    });
```

```
[2020-12-12 16:32:56] local.DEBUG: C is failed  
[2020-12-12 16:32:59] local.DEBUG: A is success  
[2020-12-12 16:32:59] local.DEBUG: B is success  
[2020-12-12 16:33:01] local.DEBUG: D is success  
```

#### 処理時間が長い場合
次は、指定時間に対して、処理の時間が長い場合の挙動を見てみます。

以下が手順と期待値です。
短い場合と結果は変わりませんが、最初の処理がおわっていなくても、指定時間が経過すれば、実行可能になっていることがわかります。

1. 処理A(10秒)を実行
2. 処理B(10秒)を実行
3. 処理C(10秒)を実行 -> 失敗
4. 3秒まつがAもBも終わっていない
5. 処理D(10秒)を実行 -> 成功

```php
function run()
{
    dispatch(new LaravelRedisTest('A'));
    dispatch(new LaravelRedisTest('B'));
    dispatch(new LaravelRedisTest('C'));
    sleep(3);
    dispatch(new LaravelRedisTest('D'));
}
```

```php
$redis = LaravelRedis::connection();
$redis->throttle(self::class)
    ->allow(2)
    ->every(3)
    ->block(0)
    ->then(function () {
        sleep(10);
        echo "{$this->name} is success" . PHP_EOL;
    }, function (LimiterTimeoutException $e) {
        echo "{$this->name} is failed" . PHP_EOL;
    });
```

```
[2020-12-12 16:35:57] local.DEBUG: C is failed  
[2020-12-12 16:36:06] local.DEBUG: A is success  
[2020-12-12 16:36:06] local.DEBUG: B is success  
[2020-12-12 16:36:10] local.DEBUG: D is success  
```

## まとめ
わたしが実際に仕事で使ったのはthrottleの方ですが、funnelと比べて仕掛けや挙動がわかりづらく迷いました。
よかったら使ってみてください。

