# Redis チートシート

## 接続

```bash
# redis-cliで接続
redis-cli
redis-cli -h hostname -p 6379
redis-cli -h hostname -p 6379 -a password
redis-cli -u redis://username:password@hostname:6379/0

# データベース選択
SELECT 0  # デフォルト
SELECT 1

# 接続テスト
PING  # => PONG

# 認証
AUTH password
```

## 基本コマンド

```bash
# キーの設定・取得
SET key value
GET key
GETSET key value  # 値を設定して古い値を返す

# キーの削除
DEL key
DEL key1 key2 key3
UNLINK key  # 非同期削除

# キーの存在確認
EXISTS key
EXISTS key1 key2  # 複数チェック

# キーの一覧
KEYS *
KEYS user:*
KEYS user:?00
SCAN 0 MATCH user:* COUNT 100  # より安全

# キーのリネーム
RENAME oldkey newkey
RENAMENX oldkey newkey  # 新キーが存在しなければ

# キーのタイプ確認
TYPE key

# TTL（有効期限）
EXPIRE key 3600  # 3600秒後に削除
EXPIREAT key 1672531200  # Unix時刻で指定
TTL key  # 残り秒数
PTTL key  # 残りミリ秒
PERSIST key  # 有効期限を削除

# データベース操作
FLUSHDB  # 現在のDBを全削除
FLUSHALL  # すべてのDBを全削除
DBSIZE  # キー数
SELECT 1  # DB切り替え
```

## 文字列（String）

```bash
# 基本操作
SET key value
SET key value EX 3600  # 有効期限付き（秒）
SET key value PX 3600000  # ミリ秒
SET key value NX  # キーが存在しなければ
SET key value XX  # キーが存在すれば
SETEX key 3600 value  # SET + EXPIRE
SETNX key value  # SET NXと同じ

GET key
MGET key1 key2 key3  # 複数取得
MSET key1 value1 key2 value2  # 複数設定

# 追加
APPEND key value

# 部分文字列
GETRANGE key 0 5
SETRANGE key 6 "new text"

# 長さ
STRLEN key

# 数値操作
INCR key  # 1増加
INCRBY key 5  # 5増加
INCRBYFLOAT key 1.5
DECR key  # 1減少
DECRBY key 5  # 5減少
```

## リスト（List）

```bash
# 追加
LPUSH key value1 value2  # 左端に追加
RPUSH key value1 value2  # 右端に追加
LPUSHX key value  # キーが存在すれば
RPUSHX key value

# 取得
LRANGE key 0 -1  # すべて
LRANGE key 0 9  # 最初の10個
LINDEX key 0  # インデックス指定

# 削除
LPOP key  # 左端を削除して返す
RPOP key  # 右端を削除して返す
LREM key count value  # 値で削除
LTRIM key 0 99  # 範囲外を削除

# 長さ
LLEN key

# 挿入
LINSERT key BEFORE pivot value
LINSERT key AFTER pivot value

# 設定
LSET key index value

# ブロッキング
BLPOP key1 key2 timeout
BRPOP key1 key2 timeout

# リスト間移動
RPOPLPUSH source destination
BRPOPLPUSH source destination timeout
```

## セット（Set）

```bash
# 追加
SADD key member1 member2

# 削除
SREM key member1 member2
SPOP key  # ランダムに削除して返す
SPOP key 3  # 3個削除

# 取得
SMEMBERS key  # すべて
SISMEMBER key member  # 存在確認
SRANDMEMBER key  # ランダムに取得
SRANDMEMBER key 3  # 3個取得

# 数
SCARD key

# 集合演算
SINTER key1 key2  # 積集合
SUNION key1 key2  # 和集合
SDIFF key1 key2  # 差集合

# 集合演算して保存
SINTERSTORE dest key1 key2
SUNIONSTORE dest key1 key2
SDIFFSTORE dest key1 key2

# 移動
SMOVE source dest member
```

## ソート済みセット（Sorted Set / ZSet）

```bash
# 追加
ZADD key score1 member1 score2 member2
ZADD key NX score member  # 存在しなければ
ZADD key XX score member  # 存在すれば
ZADD key GT score member  # スコアが大きければ
ZADD key LT score member  # スコアが小さければ

# 削除
ZREM key member1 member2
ZREMRANGEBYRANK key 0 2  # ランク範囲で削除
ZREMRANGEBYSCORE key min max  # スコア範囲で削除

# 取得（スコア昇順）
ZRANGE key 0 -1  # すべて
ZRANGE key 0 9  # 最初の10個
ZRANGE key 0 -1 WITHSCORES  # スコア付き

# 取得（スコア降順）
ZREVRANGE key 0 -1
ZREVRANGE key 0 9 WITHSCORES

# スコア範囲で取得
ZRANGEBYSCORE key min max
ZRANGEBYSCORE key 0 100 WITHSCORES LIMIT 0 10

# ランク取得
ZRANK key member  # 昇順でのランク
ZREVRANK key member  # 降順でのランク

# スコア取得
ZSCORE key member

# 数
ZCARD key
ZCOUNT key min max  # スコア範囲の数

# スコア増減
ZINCRBY key increment member

# 集合演算
ZINTERSTORE dest numkeys key1 key2
ZUNIONSTORE dest numkeys key1 key2
```

## ハッシュ（Hash）

```bash
# 設定
HSET key field value
HSET key field1 value1 field2 value2  # 複数
HSETNX key field value  # フィールドが存在しなければ
HMSET key field1 value1 field2 value2  # 非推奨（HSETを使用）

# 取得
HGET key field
HMGET key field1 field2
HGETALL key  # すべて
HKEYS key  # すべてのフィールド
HVALS key  # すべての値

# 削除
HDEL key field1 field2

# 存在確認
HEXISTS key field

# 数
HLEN key

# 数値操作
HINCRBY key field increment
HINCRBYFLOAT key field increment

# フィールド一覧
HSCAN key 0 MATCH pattern COUNT 100
```

## ビットマップ

```bash
# ビット設定
SETBIT key offset value

# ビット取得
GETBIT key offset

# ビットカウント
BITCOUNT key
BITCOUNT key start end

# ビット演算
BITOP AND destkey key1 key2
BITOP OR destkey key1 key2
BITOP XOR destkey key1 key2
BITOP NOT destkey key

# ビット位置
BITPOS key bit
BITPOS key bit start end
```

## HyperLogLog

```bash
# 追加
PFADD key element1 element2

# カウント
PFCOUNT key
PFCOUNT key1 key2

# マージ
PFMERGE destkey sourcekey1 sourcekey2
```

## ストリーム（Stream）

```bash
# 追加
XADD stream * field1 value1 field2 value2
XADD stream MAXLEN ~ 1000 * field value  # 最大長制限

# 読み取り
XREAD COUNT 10 STREAMS stream 0
XREAD BLOCK 5000 STREAMS stream $  # 新しいエントリ待ち

# 範囲取得
XRANGE stream - +
XRANGE stream 1672531200000 1672617600000
XREVRANGE stream + -

# 長さ
XLEN stream

# 削除
XDEL stream id1 id2
XTRIM stream MAXLEN ~ 1000

# コンシューマーグループ
XGROUP CREATE stream group 0
XREADGROUP GROUP group consumer COUNT 10 STREAMS stream >
XACK stream group id1 id2
```

## Pub/Sub

```bash
# パブリッシュ
PUBLISH channel message

# サブスクライブ
SUBSCRIBE channel1 channel2
PSUBSCRIBE news.*  # パターンマッチ

# アンサブスクライブ
UNSUBSCRIBE channel1
PUNSUBSCRIBE news.*

# チャンネル一覧
PUBSUB CHANNELS
PUBSUB CHANNELS news.*

# サブスクライバー数
PUBSUB NUMSUB channel1 channel2
```

## トランザクション

```bash
# トランザクション開始
MULTI

# コマンド追加
SET key1 value1
INCR key2
LPUSH key3 value

# 実行
EXEC

# キャンセル
DISCARD

# Watch（楽観的ロック）
WATCH key1 key2
MULTI
SET key1 newvalue
EXEC  # WATCHしたキーが変更されていればEXECは失敗
UNWATCH
```

## Lua スクリプト

```bash
# 実行
EVAL "return redis.call('SET', KEYS[1], ARGV[1])" 1 mykey myvalue

# スクリプト例
EVAL "
local current = redis.call('GET', KEYS[1])
if current == false then
    redis.call('SET', KEYS[1], ARGV[1])
    return 1
else
    return 0
end
" 1 mykey myvalue

# スクリプトをロード
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
# => SHA1ハッシュが返される

# SHA1で実行
EVALSHA sha1 numkeys key [key ...] arg [arg ...]

# スクリプト管理
SCRIPT EXISTS sha1 [sha1 ...]
SCRIPT FLUSH
SCRIPT KILL
```

## パイプライン

```bash
# redis-cliで
echo -e "SET key1 value1\nGET key1\nINCR counter" | redis-cli --pipe

# Pythonで
import redis
r = redis.Redis()
pipe = r.pipeline()
pipe.set('key1', 'value1')
pipe.get('key1')
pipe.incr('counter')
results = pipe.execute()
```

## 永続化

```bash
# RDB（スナップショット）
SAVE  # 同期的に保存（ブロック）
BGSAVE  # バックグラウンドで保存
LASTSAVE  # 最後の保存時刻

# AOF（Append Only File）
BGREWRITEAOF  # AOFファイルを最適化
```

## サーバー管理

```bash
# 情報表示
INFO
INFO server
INFO stats
INFO memory
INFO replication

# 設定確認
CONFIG GET *
CONFIG GET maxmemory
CONFIG GET save

# 設定変更
CONFIG SET maxmemory 2gb
CONFIG SET save "900 1 300 10"

# メモリ使用量
MEMORY USAGE key
MEMORY STATS

# クライアント一覧
CLIENT LIST
CLIENT KILL ip:port

# スロー�ログ
SLOWLOG GET 10
SLOWLOG LEN
SLOWLOG RESET

# モニター（デバッグ用）
MONITOR

# シャットダウン
SHUTDOWN SAVE
SHUTDOWN NOSAVE
```

## キー設計パターン

```bash
# ユーザー情報（Hash）
HSET user:1000 name "Alice" email "alice@example.com" age 30

# セッション（String + TTL）
SET session:abc123 "user_data" EX 3600

# カウンター（String）
INCR page:views:homepage
INCRBY user:1000:posts 1

# ランキング（Sorted Set）
ZADD leaderboard 100 user:1000
ZADD leaderboard 150 user:1001

# タグ（Set）
SADD post:1:tags "redis" "database" "nosql"

# タイムライン（List）
LPUSH user:1000:timeline "post:123"
LRANGE user:1000:timeline 0 19

# キャッシュ（String + TTL）
SET cache:api:users:all "{json data}" EX 300

# レート制限（String + TTL）
SET ratelimit:user:1000:api 0 EX 3600 NX
INCR ratelimit:user:1000:api

# 通知キュー（List）
LPUSH queue:notifications "notification_data"
BRPOP queue:notifications 0
```

## ベストプラクティス

```bash
# キー名の規則
# namespace:object:id:field
user:1000:profile
user:1000:sessions
order:5000:items

# KYESの代わりにSCANを使用
SCAN 0 MATCH user:* COUNT 100

# 大きなコレクションの分割
# 1つのハッシュに100万件ではなく
# 複数のハッシュに分割
HSET user:1000:fields:1 field1 value1
HSET user:1000:fields:2 field2 value2

# TTLの活用
SET cache:key value EX 3600

# パイプラインで複数コマンドを一度に
# ネットワークRTT削減

# Luaスクリプトでアトミック操作
# MULTI/EXECよりも効率的な場合が多い
```

## Python（redis-py）

```python
import redis

# 接続
r = redis.Redis(host='localhost', port=6379, db=0)
r = redis.from_url('redis://localhost:6379/0')

# 基本操作
r.set('key', 'value')
r.get('key')  # => b'value'
r.get('key').decode('utf-8')  # => 'value'

# TTL
r.setex('key', 3600, 'value')
r.expire('key', 3600)

# リスト
r.lpush('mylist', 'value1', 'value2')
r.lrange('mylist', 0, -1)

# ハッシュ
r.hset('user:1000', 'name', 'Alice')
r.hset('user:1000', mapping={'email': 'alice@example.com', 'age': 30})
r.hget('user:1000', 'name')
r.hgetall('user:1000')

# セット
r.sadd('myset', 'member1', 'member2')
r.smembers('myset')

# ソート済みセット
r.zadd('leaderboard', {'user1': 100, 'user2': 150})
r.zrange('leaderboard', 0, -1, withscores=True)

# パイプライン
pipe = r.pipeline()
pipe.set('key1', 'value1')
pipe.get('key1')
pipe.incr('counter')
results = pipe.execute()

# Pub/Sub
pubsub = r.pubsub()
pubsub.subscribe('channel')
for message in pubsub.listen():
    print(message)

# 接続プール
pool = redis.ConnectionPool(host='localhost', port=6379, db=0)
r = redis.Redis(connection_pool=pool)
```

## Node.js（ioredis）

```javascript
const Redis = require('ioredis');

// 接続
const redis = new Redis({
  host: 'localhost',
  port: 6379,
  db: 0
});

// 基本操作
await redis.set('key', 'value');
const value = await redis.get('key');

// TTL
await redis.setex('key', 3600, 'value');

// ハッシュ
await redis.hset('user:1000', 'name', 'Alice');
await redis.hmset('user:1000', { email: 'alice@example.com', age: 30 });
const name = await redis.hget('user:1000', 'name');
const user = await redis.hgetall('user:1000');

// パイプライン
const pipeline = redis.pipeline();
pipeline.set('key1', 'value1');
pipeline.get('key1');
pipeline.incr('counter');
const results = await pipeline.exec();

// Pub/Sub
const subscriber = new Redis();
subscriber.subscribe('channel');
subscriber.on('message', (channel, message) => {
  console.log(channel, message);
});

await redis.publish('channel', 'Hello');

// クラスタ
const cluster = new Redis.Cluster([
  { host: 'localhost', port: 6379 },
  { host: 'localhost', port: 6380 }
]);
```

## よくある使用例

```bash
# セッション管理
SET session:abc123 "{user_id: 1000}" EX 1800

# キャッシュ
SET cache:user:1000 "{json}" EX 300

# レート制限（1時間に100リクエスト）
INCR ratelimit:user:1000:20240101:12
EXPIRE ratelimit:user:1000:20240101:12 3600

# 分散ロック
SET lock:resource "token" NX EX 30
# 処理
DEL lock:resource

# カウンター
INCR counter:page:views
INCRBY counter:user:1000:points 10

# ランキング
ZADD leaderboard 100 user:1000
ZINCRBY leaderboard 10 user:1000
ZREVRANGE leaderboard 0 9 WITHSCORES

# 最近のアイテム
LPUSH recent:items item_id
LTRIM recent:items 0 99

# ユニークビジター（HyperLogLog）
PFADD unique:visitors:20240101 user1 user2
PFCOUNT unique:visitors:20240101
```
