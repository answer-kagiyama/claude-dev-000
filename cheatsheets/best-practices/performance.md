# Performance 最適化ベストプラクティス

## フロントエンド最適化

### バンドルサイズ削減

```javascript
// ✅ Tree Shaking
// 使わない部分は含まれない
import { debounce } from 'lodash-es';

// ❌ 全体をインポート
import _ from 'lodash';

// ✅ Code Splitting（Lazy Loading）
const HeavyComponent = React.lazy(() => import('./HeavyComponent'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <HeavyComponent />
    </Suspense>
  );
}

// ✅ Dynamic Import
button.addEventListener('click', async () => {
  const module = await import('./heavy-feature.js');
  module.initialize();
});
```

### リソース最適化

```html
<!-- ✅ 画像最適化 -->
<img
  src="image.webp"
  srcset="image-320w.webp 320w, image-640w.webp 640w, image-1024w.webp 1024w"
  sizes="(max-width: 640px) 100vw, 640px"
  alt="Description"
  loading="lazy"
  decoding="async"
/>

<!-- ✅ 優先度の高いリソース -->
<link rel="preload" href="critical.css" as="style" />
<link rel="preload" href="hero-image.webp" as="image" />

<!-- ✅ 優先度の低いリソース -->
<link rel="prefetch" href="next-page.js" />

<!-- ✅ DNS Prefetch -->
<link rel="dns-prefetch" href="https://api.example.com" />
<link rel="preconnect" href="https://api.example.com" />

<!-- ❌ Render Blocking -->
<script src="large-script.js"></script>

<!-- ✅ Async / Defer -->
<script src="analytics.js" async></script>
<script src="app.js" defer></script>
```

### CSS最適化

```css
/* ✅ Critical CSS をインライン化 */
<style>
  /* Above the fold の CSS のみ */
  .header { /* ... */ }
  .hero { /* ... */ }
</style>

/* ❌ 複雑なセレクタ */
div > ul > li > a:hover {
  color: red;
}

/* ✅ シンプルなセレクタ */
.nav-link:hover {
  color: red;
}

/* ✅ CSS Containment */
.card {
  contain: layout style paint;
}

/* ✅ will-change（慎重に使用） */
.animated {
  will-change: transform;
}
```

### レンダリング最適化

```javascript
// ✅ Virtual Scrolling（大量リスト）
import { FixedSizeList } from 'react-window';

function LargeList({ items }) {
  return (
    <FixedSizeList
      height={600}
      itemCount={items.length}
      itemSize={50}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>{items[index]}</div>
      )}
    </FixedSizeList>
  );
}

// ✅ Debounce / Throttle
import { debounce } from 'lodash-es';

const handleSearch = debounce((query) => {
  // API call
}, 300);

// ✅ React.memo（不要な再レンダリング防止）
const ExpensiveComponent = React.memo(({ data }) => {
  return <div>{/* Heavy rendering */}</div>;
});

// ✅ useMemo / useCallback
const memoizedValue = useMemo(() => {
  return expensiveCalculation(data);
}, [data]);

const memoizedCallback = useCallback(() => {
  doSomething(a, b);
}, [a, b]);
```

## バックエンド最適化

### N+1問題の解決

```javascript
// ❌ N+1 Problem
const users = await User.findAll();
for (const user of users) {
  user.posts = await Post.findAll({ where: { userId: user.id } });
}
// 1 + N 回のクエリ

// ✅ Eager Loading
const users = await User.findAll({
  include: [{ model: Post }]
});
// 1回のクエリ（JOIN）

// ✅ DataLoader（GraphQL）
const postLoader = new DataLoader(async (userIds) => {
  const posts = await Post.findAll({
    where: { userId: { [Op.in]: userIds } }
  });
  return userIds.map(id => posts.filter(p => p.userId === id));
});
```

### 非同期処理

```javascript
// ❌ 直列処理（遅い）
const user = await getUser(userId);
const posts = await getPosts(userId);
const comments = await getComments(userId);

// ✅ 並列処理（速い）
const [user, posts, comments] = await Promise.all([
  getUser(userId),
  getPosts(userId),
  getComments(userId)
]);

// ✅ バッチ処理
async function processUsers(userIds) {
  const BATCH_SIZE = 100;

  for (let i = 0; i < userIds.length; i += BATCH_SIZE) {
    const batch = userIds.slice(i, i + BATCH_SIZE);
    await Promise.all(batch.map(id => processUser(id)));
  }
}

// ✅ Worker Threads（CPU集約的処理）
const { Worker } = require('worker_threads');

function runHeavyTask(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./worker.js', { workerData: data });
    worker.on('message', resolve);
    worker.on('error', reject);
  });
}
```

### ストリーミング

```javascript
// ✅ ストリーム処理（大きなファイル）
const fs = require('fs');
const zlib = require('zlib');

// ❌ メモリに全て読み込む
const data = fs.readFileSync('large-file.txt');
const compressed = zlib.gzipSync(data);
fs.writeFileSync('output.gz', compressed);

// ✅ ストリームで処理
fs.createReadStream('large-file.txt')
  .pipe(zlib.createGzip())
  .pipe(fs.createWriteStream('output.gz'));

// ✅ HTTP ストリーミング
app.get('/download', (req, res) => {
  res.setHeader('Content-Type', 'text/csv');
  res.setHeader('Content-Disposition', 'attachment; filename=data.csv');

  const stream = getDataStream();
  stream.pipe(res);
});
```

## データベース最適化

### インデックス

```sql
-- ✅ 複合インデックス（順序が重要）
CREATE INDEX idx_user_status_created ON users(status, created_at);

-- 効果的
WHERE status = 'active' AND created_at > '2024-01-01'
WHERE status = 'active'

-- 非効果的
WHERE created_at > '2024-01-01'

-- ✅ カバリングインデックス
CREATE INDEX idx_covering ON orders(customer_id, created_at)
INCLUDE (status, total_amount);

-- インデックスのみでクエリ解決
SELECT status, total_amount
FROM orders
WHERE customer_id = 123
ORDER BY created_at DESC;

-- ✅ 部分インデックス
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;
```

### クエリ最適化

```sql
-- ❌ SELECT *
SELECT * FROM users WHERE id = 123;

-- ✅ 必要なカラムのみ
SELECT id, name, email FROM users WHERE id = 123;

-- ❌ OFFSET（大きな値で遅い）
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 10000;

-- ✅ Keyset Pagination
SELECT * FROM posts
WHERE created_at < '2024-01-01 00:00:00'
ORDER BY created_at DESC
LIMIT 20;

-- ❌ サブクエリ（相関サブクエリ）
SELECT u.name,
  (SELECT COUNT(*) FROM posts WHERE user_id = u.id) as post_count
FROM users u;

-- ✅ JOIN
SELECT u.name, COUNT(p.id) as post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
GROUP BY u.id, u.name;

-- ✅ EXISTS（大きなテーブル）
SELECT * FROM users u
WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```

### コネクションプール

```javascript
// ✅ 適切なプールサイズ
const pool = new Pool({
  max: 20,  // 最大接続数
  min: 5,   // 最小接続数（アイドル接続）
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000
});

// 計算式: max = (core_count * 2) + effective_spindle_count
// CPU 4コア + SSD → max = (4 * 2) + 1 = 9
```

## キャッシング

### 階層的キャッシュ

```javascript
// ✅ L1: メモリキャッシュ（最速）
const cache = new Map();

function getCachedData(key) {
  if (cache.has(key)) {
    return cache.get(key);
  }

  const data = fetchData(key);
  cache.set(key, data);
  return data;
}

// ✅ L2: Redis（共有キャッシュ）
async function getData(key) {
  // L1 チェック
  if (cache.has(key)) {
    return cache.get(key);
  }

  // L2 チェック
  const cached = await redis.get(key);
  if (cached) {
    const data = JSON.parse(cached);
    cache.set(key, data);  // L1 に保存
    return data;
  }

  // データベースから取得
  const data = await db.query(key);

  // キャッシュに保存
  await redis.setex(key, 3600, JSON.stringify(data));
  cache.set(key, data);

  return data;
}
```

### キャッシュ戦略

```javascript
// ✅ Cache-Aside（Lazy Loading）
async function getUser(id) {
  const cached = await redis.get(`user:${id}`);
  if (cached) return JSON.parse(cached);

  const user = await db.users.findById(id);
  await redis.setex(`user:${id}`, 3600, JSON.stringify(user));
  return user;
}

// ✅ Write-Through
async function updateUser(id, data) {
  const user = await db.users.update(id, data);
  await redis.setex(`user:${id}`, 3600, JSON.stringify(user));
  return user;
}

// ✅ Write-Behind（非同期書き込み）
const writeQueue = [];

async function updateUser(id, data) {
  await redis.setex(`user:${id}`, 3600, JSON.stringify(data));
  writeQueue.push({ id, data });
  return data;
}

setInterval(async () => {
  const batch = writeQueue.splice(0, 100);
  await db.users.bulkUpdate(batch);
}, 5000);

// ✅ Cache Invalidation
async function deleteUser(id) {
  await db.users.delete(id);
  await redis.del(`user:${id}`);
}
```

### HTTP キャッシュ

```javascript
// ✅ ETag（条件付きGET）
app.get('/api/users/:id', async (req, res) => {
  const user = await getUser(req.params.id);
  const etag = `"${hashObject(user)}"`;

  if (req.headers['if-none-match'] === etag) {
    return res.status(304).end();  // Not Modified
  }

  res.setHeader('ETag', etag);
  res.setHeader('Cache-Control', 'private, max-age=3600');
  res.json(user);
});

// ✅ CDN キャッシュ
res.setHeader('Cache-Control', 'public, max-age=86400, s-maxage=604800');
// ブラウザ: 1日、CDN: 7日
```

## ネットワーク最適化

### HTTP/2

```javascript
// ✅ HTTP/2 Server Push
const http2 = require('http2');

const server = http2.createSecureServer({
  key: fs.readFileSync('key.pem'),
  cert: fs.readFileSync('cert.pem')
});

server.on('stream', (stream, headers) => {
  if (headers[':path'] === '/') {
    // HTML を送信
    stream.respond({
      'content-type': 'text/html',
      ':status': 200
    });
    stream.end('<html>...</html>');

    // CSS/JS を Push
    stream.pushStream({ ':path': '/style.css' }, (err, pushStream) => {
      pushStream.respond({ 'content-type': 'text/css' });
      pushStream.end(cssContent);
    });
  }
});
```

### Compression

```javascript
// ✅ Gzip / Brotli
const compression = require('compression');

app.use(compression({
  filter: (req, res) => {
    if (req.headers['x-no-compression']) {
      return false;
    }
    return compression.filter(req, res);
  },
  level: 6  // 圧縮レベル（1-9）
}));

// ✅ 静的ファイルは事前圧縮
// build時に.gzファイルを生成
// nginx で gzip_static on;
```

### API最適化

```javascript
// ✅ GraphQL（必要なデータのみ）
query {
  user(id: "123") {
    name
    email
    posts {
      title
    }
  }
}

// ✅ Batch API
POST /api/batch
{
  "requests": [
    { "method": "GET", "url": "/users/1" },
    { "method": "GET", "url": "/users/2" },
    { "method": "GET", "url": "/posts/1" }
  ]
}

// ✅ Pagination
GET /api/users?cursor=eyJpZCI6MTIzfQ&limit=20
```

## メモリ管理

### メモリリーク防止

```javascript
// ❌ メモリリーク
const cache = new Map();

setInterval(() => {
  const data = fetchData();
  cache.set(Date.now(), data);  // 無限に増加
}, 1000);

// ✅ サイズ制限付きキャッシュ
class LRUCache {
  constructor(maxSize = 100) {
    this.cache = new Map();
    this.maxSize = maxSize;
  }

  set(key, value) {
    if (this.cache.size >= this.maxSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);  // 最古のエントリを削除
    }
    this.cache.set(key, value);
  }
}

// ✅ WeakMap（自動GC）
const privateData = new WeakMap();

class User {
  constructor(name) {
    privateData.set(this, { name });
  }
}
// User インスタンスがGCされると、privateDataも自動的に削除

// ✅ リソースのクリーンアップ
class Connection {
  constructor() {
    this.socket = createSocket();
  }

  close() {
    this.socket.close();
    this.socket = null;  // 参照を削除
  }
}
```

### ガベージコレクション

```javascript
// ✅ 大きな配列の処理
function processLargeArray(arr) {
  const CHUNK_SIZE = 1000;

  for (let i = 0; i < arr.length; i += CHUNK_SIZE) {
    const chunk = arr.slice(i, i + CHUNK_SIZE);
    processChunk(chunk);

    // GCの機会を与える
    if (i % 10000 === 0) {
      await new Promise(resolve => setImmediate(resolve));
    }
  }
}

// Node.js GC設定
// --max-old-space-size=4096  # ヒープサイズ 4GB
// --expose-gc  # 手動GC有効化
```

## コード最適化

### アルゴリズム

```javascript
// ❌ O(n²) - 遅い
function findDuplicates(arr) {
  const duplicates = [];
  for (let i = 0; i < arr.length; i++) {
    for (let j = i + 1; j < arr.length; j++) {
      if (arr[i] === arr[j]) {
        duplicates.push(arr[i]);
      }
    }
  }
  return duplicates;
}

// ✅ O(n) - 速い
function findDuplicates(arr) {
  const seen = new Set();
  const duplicates = new Set();

  for (const item of arr) {
    if (seen.has(item)) {
      duplicates.add(item);
    }
    seen.add(item);
  }

  return Array.from(duplicates);
}
```

### データ構造

```javascript
// ❌ 配列で検索（O(n)）
const users = [{ id: 1 }, { id: 2 }, { id: 3 }];
const user = users.find(u => u.id === 2);

// ✅ Map で検索（O(1)）
const users = new Map([
  [1, { id: 1 }],
  [2, { id: 2 }],
  [3, { id: 3 }]
]);
const user = users.get(2);

// ✅ Set（重複削除）
const uniqueIds = [...new Set([1, 2, 2, 3, 3, 3])];  // [1, 2, 3]
```

## モニタリング・計測

### パフォーマンス計測

```javascript
// ✅ Performance API
const start = performance.now();
await heavyOperation();
const duration = performance.now() - start;
console.log(`Duration: ${duration}ms`);

// ✅ Performance Observer
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(`${entry.name}: ${entry.duration}ms`);
  }
});

observer.observe({ entryTypes: ['measure', 'navigation'] });

performance.mark('start');
await operation();
performance.mark('end');
performance.measure('operation', 'start', 'end');

// ✅ Node.js Profiler
node --prof app.js
node --prof-process isolate-*.log > processed.txt

// ✅ Chrome DevTools
// Performance タブでプロファイリング
// Memory タブでメモリリーク検出
```

### APM（Application Performance Monitoring）

```javascript
// ✅ New Relic
const newrelic = require('newrelic');

app.get('/api/users', async (req, res) => {
  const transaction = newrelic.getTransaction();
  transaction.acceptDistributedTraceHeaders('HTTP', req.headers);

  const users = await getUsers();
  res.json(users);
});

// ✅ DataDog
const tracer = require('dd-trace').init();

app.get('/api/users', async (req, res) => {
  const span = tracer.startSpan('get_users');

  try {
    const users = await getUsers();
    res.json(users);
  } finally {
    span.finish();
  }
});
```

### メトリクス

```javascript
// ✅ Prometheus
const client = require('prom-client');

const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code']
});

app.use((req, res, next) => {
  const start = Date.now();

  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    httpRequestDuration
      .labels(req.method, req.route?.path, res.statusCode)
      .observe(duration);
  });

  next();
});

// メトリクスエンドポイント
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});
```

## チェックリスト

### フロントエンド
- [ ] バンドルサイズ最適化（Tree Shaking, Code Splitting）
- [ ] 画像最適化（WebP, Lazy Loading）
- [ ] Critical CSS インライン化
- [ ] リソースのpreload/prefetch
- [ ] レンダリング最適化（Virtual Scrolling, React.memo）

### バックエンド
- [ ] N+1問題の解決
- [ ] 非同期処理の並列化
- [ ] ストリーミング処理
- [ ] コネクションプール設定
- [ ] 適切なアルゴリズム・データ構造

### データベース
- [ ] 適切なインデックス
- [ ] クエリ最適化（EXPLAIN ANALYZE）
- [ ] Keyset Pagination
- [ ] コネクションプール

### キャッシング
- [ ] 階層的キャッシュ（メモリ + Redis）
- [ ] 適切なTTL設定
- [ ] Cache Invalidation戦略
- [ ] HTTP キャッシュヘッダー

### ネットワーク
- [ ] HTTP/2使用
- [ ] Compression（Gzip/Brotli）
- [ ] CDN活用
- [ ] API最適化

### モニタリング
- [ ] パフォーマンス計測
- [ ] APM導入
- [ ] メトリクス収集
- [ ] アラート設定

## 参考リソース

- [Web.dev Performance](https://web.dev/performance/)
- [Core Web Vitals](https://web.dev/vitals/)
- [Node.js Best Practices](https://github.com/goldbergyoni/nodebestpractices)
- [Database Performance Tips](https://use-the-index-luke.com/)
