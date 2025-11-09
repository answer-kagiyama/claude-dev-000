# Database 設計ベストプラクティス

## スキーマ設計

### 正規化

```sql
-- ❌ 非正規化（冗長性あり）
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_name VARCHAR(100),    -- 重複
    customer_email VARCHAR(100),   -- 重複
    customer_phone VARCHAR(20),    -- 重複
    product_name VARCHAR(100),     -- 重複
    product_price DECIMAL(10,2),   -- 重複
    quantity INT
);

-- ✅ 第3正規形
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NOT NULL CHECK (price >= 0),
    stock INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INT NOT NULL REFERENCES customers(id),
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id INT NOT NULL REFERENCES products(id),
    quantity INT NOT NULL CHECK (quantity > 0),
    price DECIMAL(10,2) NOT NULL,  -- 購入時の価格を保存
    UNIQUE(order_id, product_id)
);
```

### 戦略的非正規化

```sql
-- ✅ パフォーマンスのための非正規化
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INT NOT NULL REFERENCES customers(id),
    total_amount DECIMAL(10,2) NOT NULL,  -- 集計値をキャッシュ
    item_count INT NOT NULL,              -- 集計値をキャッシュ
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- トリガーで整合性を維持
CREATE OR REPLACE FUNCTION update_order_totals()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE orders
    SET
        total_amount = (
            SELECT COALESCE(SUM(quantity * price), 0)
            FROM order_items
            WHERE order_id = NEW.order_id
        ),
        item_count = (
            SELECT COALESCE(COUNT(*), 0)
            FROM order_items
            WHERE order_id = NEW.order_id
        )
    WHERE id = NEW.order_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER order_items_update
AFTER INSERT OR UPDATE OR DELETE ON order_items
FOR EACH ROW EXECUTE FUNCTION update_order_totals();
```

## データ型選択

```sql
-- ✅ 適切なデータ型
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),  -- セキュアなID
    email VARCHAR(255) NOT NULL,                    -- 適切なサイズ
    age SMALLINT CHECK (age >= 0 AND age <= 150),  -- 範囲制限
    is_active BOOLEAN DEFAULT true,                 -- フラグ
    balance NUMERIC(10,2),                          -- 金額（正確性重視）
    last_login TIMESTAMP WITH TIME ZONE,           -- タイムゾーン付き
    metadata JSONB,                                 -- 柔軟なデータ
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ❌ 避けるべき
CREATE TABLE bad_users (
    id INT,                          -- ❌ AUTO_INCREMENT が好ましい
    email TEXT,                      -- ❌ VARCHAR(255) が好ましい
    age INT,                         -- ❌ SMALLINT で十分
    is_active VARCHAR(5),            -- ❌ BOOLEAN を使うべき
    balance FLOAT,                   -- ❌ 金額には不正確
    last_login DATE,                 -- ❌ 時刻情報が失われる
    metadata TEXT                    -- ❌ JSONB が好ましい
);
```

## インデックス設計

### 基本的なインデックス

```sql
-- ✅ PRIMARY KEY（自動的にインデックス作成）
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,  -- UNIQUE も自動的にインデックス
    name VARCHAR(100) NOT NULL
);

-- ✅ 頻繁に検索されるカラム
CREATE INDEX idx_users_email ON users(email);

-- ✅ 外部キー
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

-- ✅ 複合インデックス（順序が重要）
CREATE INDEX idx_orders_customer_status ON orders(customer_id, status);
-- WHERE customer_id = ? AND status = ? ← 効果的
-- WHERE customer_id = ? ← 効果的（先頭カラムのみでも使用可能）
-- WHERE status = ? ← 非効果的

-- ✅ 部分インデックス（条件付きインデックス）
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;

-- ✅ 式インデックス
CREATE INDEX idx_users_lower_email ON users(LOWER(email));
```

### インデックス戦略

```sql
-- ✅ カバリングインデックス（INCLUDE）
CREATE INDEX idx_orders_covering ON orders(customer_id, created_at)
INCLUDE (status, total_amount);
-- SELECT status, total_amount WHERE customer_id = ? はインデックスのみで解決

-- ✅ B-Tree（デフォルト、範囲検索に適する）
CREATE INDEX idx_created_at ON orders(created_at);

-- ✅ GiN（JSONB、配列、全文検索）
CREATE INDEX idx_metadata ON users USING GIN(metadata);

-- ✅ GiST（空間データ、範囲型）
CREATE INDEX idx_location ON stores USING GIST(location);

-- ✅ Hash（完全一致検索のみ）
CREATE INDEX idx_hash_email ON users USING HASH(email);
```

### インデックスの注意点

```sql
-- ❌ 過剰なインデックス
CREATE INDEX idx_user_id ON users(id);        -- ❌ PRIMARY KEY で十分
CREATE INDEX idx_email1 ON users(email);      -- ❌
CREATE INDEX idx_email2 ON users(LOWER(email)); -- ❌ 重複

-- ❌ 低選択性のカラム
CREATE INDEX idx_gender ON users(gender);     -- ❌ 値が2-3種類のみ
CREATE INDEX idx_is_active ON users(is_active); -- ❌ BOOLEAN

-- ✅ 部分インデックスなら有効
CREATE INDEX idx_inactive_users ON users(id) WHERE is_active = false;
```

## クエリ最適化

### N+1問題の回避

```sql
-- ❌ N+1 問題
-- アプリケーションで users をループして orders を取得
SELECT * FROM users;  -- 1回
SELECT * FROM orders WHERE user_id = ?;  -- N回

-- ✅ JOIN を使用
SELECT
    u.*,
    o.id as order_id,
    o.total_amount,
    o.created_at as order_date
FROM users u
LEFT JOIN orders o ON u.id = o.customer_id
WHERE u.is_active = true;

-- ✅ サブクエリで集計
SELECT
    u.*,
    (SELECT COUNT(*) FROM orders WHERE customer_id = u.id) as order_count,
    (SELECT SUM(total_amount) FROM orders WHERE customer_id = u.id) as total_spent
FROM users u
WHERE u.is_active = true;

-- ✅ CTEでクリーンに
WITH order_stats AS (
    SELECT
        customer_id,
        COUNT(*) as order_count,
        SUM(total_amount) as total_spent
    FROM orders
    GROUP BY customer_id
)
SELECT
    u.*,
    COALESCE(os.order_count, 0) as order_count,
    COALESCE(os.total_spent, 0) as total_spent
FROM users u
LEFT JOIN order_stats os ON u.id = os.customer_id
WHERE u.is_active = true;
```

### ページネーション

```sql
-- ❌ OFFSET は大きな値で遅い
SELECT * FROM orders
ORDER BY created_at DESC
LIMIT 20 OFFSET 100000;  -- 遅い

-- ✅ カーソルベース（Keyset Pagination）
SELECT * FROM orders
WHERE created_at < '2024-01-01 00:00:00'
ORDER BY created_at DESC
LIMIT 20;

-- ✅ 複合キーでのカーソル
SELECT * FROM orders
WHERE (created_at, id) < ('2024-01-01 00:00:00', 12345)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

### EXPLAIN ANALYZE

```sql
-- クエリプランを確認
EXPLAIN ANALYZE
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.customer_id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id, u.name
HAVING COUNT(o.id) > 5;

-- 確認ポイント
-- - Seq Scan（全件走査）→ Index Scan に改善できないか
-- - 実行時間（actual time）
-- - rows（処理行数）
-- - buffers（I/O量）
```

### インデックスヒント

```sql
-- ✅ 部分インデックスの活用
CREATE INDEX idx_recent_orders ON orders(created_at)
WHERE created_at > CURRENT_DATE - INTERVAL '30 days';

-- ✅ 統計情報の更新
ANALYZE orders;
ANALYZE users;

-- ✅ 自動vacuum設定
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.1,
    autovacuum_analyze_scale_factor = 0.05
);
```

## トランザクション

### ACID特性

```sql
-- ✅ Atomicity（原子性）
BEGIN;
INSERT INTO orders (customer_id, total_amount) VALUES (1, 1000);
INSERT INTO order_items (order_id, product_id, quantity, price)
VALUES (LASTVAL(), 1, 2, 500);
UPDATE products SET stock = stock - 2 WHERE id = 1;
COMMIT;  -- すべて成功 or すべて失敗

-- ✅ Isolation（分離性）
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- 完全な分離（最も厳格、パフォーマンス低下）
COMMIT;

BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- 読み取りデータは一貫性保証（PostgreSQL推奨）
COMMIT;

BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- コミット済みデータのみ読み取り（デフォルト）
COMMIT;
```

### デッドロック対策

```sql
-- ❌ デッドロックが発生しやすい
-- Transaction 1
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- Transaction 2（同時実行）
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;
UPDATE accounts SET balance = balance + 50 WHERE id = 1;
COMMIT;

-- ✅ ロック順序を統一
-- すべてのトランザクションで id の昇順でロック
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- 小さい id
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- 大きい id
COMMIT;

-- ✅ SELECT FOR UPDATE でロック明示
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
-- ロック取得後に更新
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

### 楽観的ロック

```sql
-- ✅ バージョン番号で楽観的ロック
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    stock INT NOT NULL,
    version INT NOT NULL DEFAULT 0  -- バージョン番号
);

-- 更新時にバージョンをチェック
UPDATE products
SET
    stock = stock - 1,
    version = version + 1
WHERE id = 123 AND version = 5;  -- 現在のバージョンと一致する場合のみ更新

-- アプリケーション側で更新行数をチェック
-- 0 行なら他のトランザクションが先に更新済み → リトライ
```

## マイグレーション

### バージョン管理

```sql
-- ✅ マイグレーションファイル名
-- 001_create_users_table.sql
-- 002_add_email_index.sql
-- 003_add_user_roles.sql

-- 001_create_users_table.sql
-- Up Migration
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Down Migration (別ファイルまたは同一ファイル)
-- DROP TABLE users;
```

### ゼロダウンタイム

```sql
-- ✅ カラム追加（ゼロダウンタイム）
-- Step 1: NULL許容で追加
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Step 2: アプリケーションデプロイ（新カラム使用開始）

-- Step 3: データ移行
UPDATE users SET phone = legacy_phone WHERE phone IS NULL;

-- Step 4: NOT NULL 制約追加
ALTER TABLE users ALTER COLUMN phone SET NOT NULL;

-- ❌ 危険：一気に NOT NULL で追加
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NOT NULL;  -- ロック時間が長い
```

### インデックス作成

```sql
-- ❌ 通常の CREATE INDEX（テーブルロック）
CREATE INDEX idx_users_email ON users(email);

-- ✅ CONCURRENTLY（ロックなし）
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);

-- ⚠️ CONCURRENTLY の注意点
-- - トランザクション内で実行不可
-- - 失敗した場合、無効なインデックスが残る
-- - 削除が必要: DROP INDEX CONCURRENTLY idx_users_email;
```

## パフォーマンス

### コネクションプール

```javascript
// ✅ コネクションプール設定（Node.js pg）
const { Pool } = require('pg');

const pool = new Pool({
  host: 'localhost',
  database: 'mydb',
  user: 'user',
  password: 'password',
  max: 20,                // 最大接続数
  min: 5,                 // 最小接続数
  idleTimeoutMillis: 30000,  // アイドル接続のタイムアウト
  connectionTimeoutMillis: 2000,  // 接続取得のタイムアウト
});

// 使用
async function getUser(id) {
  const client = await pool.connect();
  try {
    const result = await client.query('SELECT * FROM users WHERE id = $1', [id]);
    return result.rows[0];
  } finally {
    client.release();  // 必ずリリース
  }
}
```

### プリペアドステートメント

```javascript
// ✅ プリペアドステートメント（SQLインジェクション対策 + 高速化）
const result = await pool.query(
  'SELECT * FROM users WHERE email = $1 AND is_active = $2',
  ['alice@example.com', true]
);

// ❌ 文字列連結（SQLインジェクションリスク）
const email = "alice@example.com";
const query = `SELECT * FROM users WHERE email = '${email}'`;
```

### バッチ処理

```sql
-- ✅ バルクインサート
INSERT INTO users (name, email) VALUES
  ('Alice', 'alice@example.com'),
  ('Bob', 'bob@example.com'),
  ('Charlie', 'charlie@example.com');

-- ✅ COPY（さらに高速）
COPY users (name, email) FROM '/path/to/data.csv' WITH CSV HEADER;

-- ❌ 1件ずつインサート
INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com');
INSERT INTO users (name, email) VALUES ('Bob', 'bob@example.com');
```

## セキュリティ

### アクセス制御

```sql
-- ✅ 最小権限の原則
CREATE USER app_user WITH PASSWORD 'strong_password';

-- 読み取り専用ユーザー
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;

-- アプリケーション用ユーザー
GRANT SELECT, INSERT, UPDATE, DELETE ON users, orders TO app_user;

-- 特定のカラムのみ
GRANT SELECT (id, name, email) ON users TO limited_user;

-- RLS（Row Level Security）
CREATE POLICY user_isolation ON users
FOR ALL
TO app_user
USING (tenant_id = current_setting('app.current_tenant')::int);

ALTER TABLE users ENABLE ROW LEVEL SECURITY;
```

### 暗号化

```sql
-- ✅ パスワードのハッシュ化
-- アプリケーション側で bcrypt 使用（DB に保存しない）

-- ✅ 機密データの暗号化（PostgreSQL pgcrypto）
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- 暗号化して保存
INSERT INTO sensitive_data (user_id, credit_card)
VALUES (1, pgp_sym_encrypt('1234-5678-9012-3456', 'encryption_key'));

-- 復号化して取得
SELECT
    user_id,
    pgp_sym_decrypt(credit_card, 'encryption_key') as credit_card
FROM sensitive_data
WHERE user_id = 1;

-- ✅ 列レベル暗号化
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    ssn BYTEA  -- 暗号化されたデータ
);
```

## バックアップとリカバリ

### バックアップ戦略

```bash
# ✅ 論理バックアップ（pg_dump）
pg_dump -h localhost -U postgres -d mydb -F c -f backup.dump

# リストア
pg_restore -h localhost -U postgres -d mydb -c backup.dump

# ✅ 物理バックアップ（pg_basebackup）
pg_basebackup -h localhost -D /backup/data -U replication -P

# ✅ 継続的アーカイブ（WAL）
# postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'cp %p /archive/%f'

# ポイントインタイムリカバリ（PITR）
pg_restore --target-time='2024-01-01 12:00:00'
```

### レプリケーション

```sql
-- ✅ ストリーミングレプリケーション
-- プライマリ
CREATE USER replication_user REPLICATION LOGIN PASSWORD 'password';

-- postgresql.conf
wal_level = replica
max_wal_senders = 3

-- pg_hba.conf
host replication replication_user 192.168.1.0/24 md5

-- スタンバイ
pg_basebackup -h primary -D /var/lib/postgresql/data -U replication_user -P

-- recovery.conf (PostgreSQL 12+では postgresql.auto.conf)
primary_conninfo = 'host=primary port=5432 user=replication_user'
```

## モニタリング

### 重要なメトリクス

```sql
-- ✅ 実行中のクエリ
SELECT
    pid,
    now() - query_start as duration,
    state,
    query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- ✅ 遅いクエリ
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    max_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- ✅ インデックス使用状況
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,  -- インデックススキャン回数
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;

-- ✅ テーブルサイズ
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- ✅ データベース接続数
SELECT
    count(*),
    state
FROM pg_stat_activity
GROUP BY state;
```

## チェックリスト

### スキーマ設計
- [ ] 正規化は適切か（第3正規形）
- [ ] 戦略的非正規化を検討したか
- [ ] 適切なデータ型を選択したか
- [ ] 制約（NOT NULL、UNIQUE、CHECK）を設定したか
- [ ] 外部キー制約を設定したか

### インデックス
- [ ] PRIMARY KEY を設定したか
- [ ] 外部キーにインデックスを作成したか
- [ ] 頻繁に検索されるカラムにインデックスがあるか
- [ ] 複合インデックスの順序は適切か
- [ ] 過剰なインデックスはないか

### パフォーマンス
- [ ] N+1問題を回避しているか
- [ ] EXPLAIN ANALYZE でクエリプランを確認したか
- [ ] コネクションプールを使用しているか
- [ ] 適切なトランザクション分離レベルか

### セキュリティ
- [ ] SQLインジェクション対策済みか
- [ ] 最小権限の原則に従っているか
- [ ] 機密データを暗号化しているか
- [ ] パスワードをハッシュ化しているか

### 運用
- [ ] マイグレーション戦略があるか
- [ ] バックアップ戦略があるか
- [ ] モニタリング設定済みか
- [ ] ログ設定は適切か

## 参考リソース

- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Use The Index, Luke](https://use-the-index-luke.com/)
- [Database Normalization](https://en.wikipedia.org/wiki/Database_normalization)
- [The Twelve-Factor App - Backing Services](https://12factor.net/backing-services)
