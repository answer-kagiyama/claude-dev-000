# PostgreSQL チートシート

## 接続・基本操作

```bash
# psqlで接続
psql -U username -d database
psql -h hostname -p 5432 -U username -d database
psql postgresql://username:password@hostname:5432/database

# よく使うメタコマンド
\l                # データベース一覧
\c database       # データベース切り替え
\dt               # テーブル一覧
\d table_name     # テーブル構造表示
\du               # ユーザー一覧
\dn               # スキーマ一覧
\df               # 関数一覧
\dv               # ビュー一覧
\di               # インデックス一覧
\x                # 縦表示モード切り替え
\q                # 終了
\?                # ヘルプ

# ファイルから実行
\i file.sql

# 出力
\o output.txt     # 出力先設定
\o                # 画面出力に戻す

# タイミング表示
\timing on
```

## データベース操作

```sql
-- データベース作成
CREATE DATABASE mydb;
CREATE DATABASE mydb OWNER myuser ENCODING 'UTF8';

-- データベース削除
DROP DATABASE mydb;
DROP DATABASE IF EXISTS mydb;

-- データベース一覧
SELECT datname FROM pg_database;

-- 接続中のデータベース確認
SELECT current_database();
```

## テーブル操作

### テーブル作成

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    age INTEGER CHECK (age >= 0),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 制約付き
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    product_name VARCHAR(200),
    quantity INTEGER DEFAULT 1,
    price DECIMAL(10, 2),
    status VARCHAR(20) CHECK (status IN ('pending', 'completed', 'cancelled')),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- テーブルコピー
CREATE TABLE users_backup AS SELECT * FROM users;
CREATE TABLE users_copy (LIKE users INCLUDING ALL);
```

### テーブル変更

```sql
-- カラム追加
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
ALTER TABLE users ADD COLUMN IF NOT EXISTS phone VARCHAR(20);

-- カラム削除
ALTER TABLE users DROP COLUMN phone;
ALTER TABLE users DROP COLUMN IF EXISTS phone;

-- カラム名変更
ALTER TABLE users RENAME COLUMN name TO full_name;

-- カラム型変更
ALTER TABLE users ALTER COLUMN age TYPE BIGINT;

-- NOT NULL制約
ALTER TABLE users ALTER COLUMN email SET NOT NULL;
ALTER TABLE users ALTER COLUMN email DROP NOT NULL;

-- デフォルト値
ALTER TABLE users ALTER COLUMN age SET DEFAULT 0;
ALTER TABLE users ALTER COLUMN age DROP DEFAULT;

-- テーブル名変更
ALTER TABLE users RENAME TO customers;

-- 制約追加
ALTER TABLE users ADD CONSTRAINT email_unique UNIQUE (email);
ALTER TABLE users ADD CONSTRAINT age_check CHECK (age >= 0);
```

### テーブル削除

```sql
DROP TABLE users;
DROP TABLE IF EXISTS users;
DROP TABLE users CASCADE;  -- 依存オブジェクトも削除

-- 全データ削除（構造は残す）
TRUNCATE TABLE users;
TRUNCATE TABLE users RESTART IDENTITY CASCADE;
```

## データ型

```sql
-- 数値型
SMALLINT            -- 2バイト整数
INTEGER, INT        -- 4バイト整数
BIGINT              -- 8バイト整数
DECIMAL(p, s)       -- 精度pスケールsの固定小数点
NUMERIC(p, s)       -- DECIMALと同じ
REAL                -- 単精度浮動小数点
DOUBLE PRECISION    -- 倍精度浮動小数点
SERIAL              -- 自動インクリメント整数
BIGSERIAL           -- 自動インクリメント大整数

-- 文字列型
CHAR(n)             -- 固定長文字列
VARCHAR(n)          -- 可変長文字列
TEXT                -- 無制限長文字列

-- 日付・時刻型
DATE                -- 日付
TIME                -- 時刻
TIMESTAMP           -- 日時
TIMESTAMPTZ         -- タイムゾーン付き日時
INTERVAL            -- 期間

-- 真偽値
BOOLEAN             -- true/false

-- その他
JSON                -- JSON
JSONB               -- バイナリJSON（推奨）
UUID                -- UUID
ARRAY               -- 配列
BYTEA               -- バイナリデータ
```

## CRUD操作

### INSERT

```sql
-- 基本
INSERT INTO users (name, email, age)
VALUES ('Alice', 'alice@example.com', 30);

-- 複数行
INSERT INTO users (name, email, age)
VALUES
    ('Bob', 'bob@example.com', 25),
    ('Charlie', 'charlie@example.com', 35);

-- SELECTから挿入
INSERT INTO users_backup
SELECT * FROM users WHERE age > 30;

-- RETURNING句（挿入したデータを返す）
INSERT INTO users (name, email)
VALUES ('Dave', 'dave@example.com')
RETURNING id, created_at;

-- ON CONFLICT（upsert）
INSERT INTO users (email, name, age)
VALUES ('alice@example.com', 'Alice', 31)
ON CONFLICT (email)
DO UPDATE SET name = EXCLUDED.name, age = EXCLUDED.age;

-- 何もしない
INSERT INTO users (email, name)
VALUES ('alice@example.com', 'Alice')
ON CONFLICT (email) DO NOTHING;
```

### SELECT

```sql
-- 基本
SELECT * FROM users;
SELECT name, email FROM users;
SELECT DISTINCT age FROM users;

-- WHERE条件
SELECT * FROM users WHERE age > 30;
SELECT * FROM users WHERE age >= 30 AND age < 40;
SELECT * FROM users WHERE name LIKE 'A%';
SELECT * FROM users WHERE name ILIKE 'alice';  -- 大文字小文字無視
SELECT * FROM users WHERE email IS NULL;
SELECT * FROM users WHERE age IN (25, 30, 35);
SELECT * FROM users WHERE age BETWEEN 20 AND 40;

-- ORDER BY
SELECT * FROM users ORDER BY age ASC;
SELECT * FROM users ORDER BY age DESC;
SELECT * FROM users ORDER BY age DESC, name ASC;

-- LIMIT/OFFSET（ページング）
SELECT * FROM users LIMIT 10;
SELECT * FROM users LIMIT 10 OFFSET 20;
SELECT * FROM users ORDER BY id LIMIT 10 OFFSET 20;

-- 集計
SELECT COUNT(*) FROM users;
SELECT COUNT(DISTINCT age) FROM users;
SELECT AVG(age) FROM users;
SELECT MAX(age), MIN(age) FROM users;
SELECT SUM(price) FROM orders;

-- GROUP BY
SELECT age, COUNT(*) FROM users GROUP BY age;
SELECT age, COUNT(*) as count FROM users
GROUP BY age
HAVING COUNT(*) > 5
ORDER BY count DESC;

-- JOIN
SELECT u.name, o.product_name
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

SELECT u.name, o.product_name
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

SELECT u.name, o.product_name
FROM users u
RIGHT JOIN orders o ON u.id = o.user_id;

SELECT u.name, o.product_name
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;

-- サブクエリ
SELECT * FROM users
WHERE age > (SELECT AVG(age) FROM users);

SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders WHERE status = 'completed');

-- WITH（CTE: Common Table Expression）
WITH high_value_orders AS (
    SELECT user_id, SUM(price) as total
    FROM orders
    GROUP BY user_id
    HAVING SUM(price) > 1000
)
SELECT u.name, h.total
FROM users u
JOIN high_value_orders h ON u.id = h.user_id;

-- CASE式
SELECT
    name,
    age,
    CASE
        WHEN age < 20 THEN 'Teen'
        WHEN age < 40 THEN 'Adult'
        ELSE 'Senior'
    END as age_group
FROM users;
```

### UPDATE

```sql
-- 基本
UPDATE users SET age = 31 WHERE id = 1;

-- 複数カラム
UPDATE users
SET name = 'Alice Smith', age = 32
WHERE id = 1;

-- 計算
UPDATE orders SET price = price * 1.1 WHERE status = 'pending';

-- FROM句
UPDATE orders
SET status = 'cancelled'
FROM users
WHERE orders.user_id = users.id AND users.age < 18;

-- RETURNING
UPDATE users SET age = age + 1
WHERE id = 1
RETURNING *;
```

### DELETE

```sql
-- 基本
DELETE FROM users WHERE id = 1;

-- 条件
DELETE FROM users WHERE age < 18;

-- 全削除（注意）
DELETE FROM users;

-- RETURNING
DELETE FROM users WHERE id = 1 RETURNING *;
```

## インデックス

```sql
-- 作成
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_name ON users(name);
CREATE INDEX idx_users_age_name ON users(age, name);  -- 複合インデックス

-- ユニークインデックス
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);

-- 部分インデックス
CREATE INDEX idx_active_users ON users(name) WHERE active = true;

-- 式インデックス
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- 削除
DROP INDEX idx_users_email;
DROP INDEX IF EXISTS idx_users_email;

-- 再構築
REINDEX INDEX idx_users_email;
REINDEX TABLE users;

-- インデックス一覧
SELECT * FROM pg_indexes WHERE tablename = 'users';
```

## ビュー

```sql
-- 作成
CREATE VIEW active_users AS
SELECT * FROM users WHERE active = true;

-- 置き換え
CREATE OR REPLACE VIEW active_users AS
SELECT id, name, email FROM users WHERE active = true;

-- マテリアライズドビュー（実体を持つ）
CREATE MATERIALIZED VIEW user_stats AS
SELECT age, COUNT(*) as count
FROM users
GROUP BY age;

-- リフレッシュ
REFRESH MATERIALIZED VIEW user_stats;

-- 削除
DROP VIEW active_users;
DROP MATERIALIZED VIEW user_stats;
```

## トランザクション

```sql
-- 基本
BEGIN;
-- または START TRANSACTION;

INSERT INTO users (name, email) VALUES ('Test', 'test@example.com');
UPDATE users SET age = 30 WHERE name = 'Test';

COMMIT;
-- または ROLLBACK;

-- セーブポイント
BEGIN;
INSERT INTO users (name, email) VALUES ('Test1', 'test1@example.com');
SAVEPOINT sp1;
INSERT INTO users (name, email) VALUES ('Test2', 'test2@example.com');
ROLLBACK TO SAVEPOINT sp1;  -- Test2のみロールバック
COMMIT;
```

## 制約

```sql
-- PRIMARY KEY
ALTER TABLE users ADD PRIMARY KEY (id);

-- FOREIGN KEY
ALTER TABLE orders
ADD CONSTRAINT fk_user
FOREIGN KEY (user_id) REFERENCES users(id)
ON DELETE CASCADE
ON UPDATE CASCADE;

-- UNIQUE
ALTER TABLE users ADD CONSTRAINT email_unique UNIQUE (email);

-- CHECK
ALTER TABLE users ADD CONSTRAINT age_check CHECK (age >= 0 AND age <= 150);

-- NOT NULL
ALTER TABLE users ALTER COLUMN email SET NOT NULL;

-- 制約削除
ALTER TABLE users DROP CONSTRAINT email_unique;
```

## 関数・プロシージャ

```sql
-- 関数作成
CREATE OR REPLACE FUNCTION get_user_count()
RETURNS INTEGER AS $$
BEGIN
    RETURN (SELECT COUNT(*) FROM users);
END;
$$ LANGUAGE plpgsql;

-- 使用
SELECT get_user_count();

-- 引数付き関数
CREATE OR REPLACE FUNCTION get_users_by_age(min_age INTEGER)
RETURNS TABLE(id INTEGER, name VARCHAR, email VARCHAR) AS $$
BEGIN
    RETURN QUERY
    SELECT users.id, users.name, users.email
    FROM users
    WHERE users.age >= min_age;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM get_users_by_age(30);

-- 削除
DROP FUNCTION get_user_count();
```

## トリガー

```sql
-- トリガー関数作成
CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- トリガー作成
CREATE TRIGGER trigger_update_timestamp
BEFORE UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION update_timestamp();

-- 削除
DROP TRIGGER trigger_update_timestamp ON users;
```

## ユーザー・権限

```sql
-- ユーザー作成
CREATE USER myuser WITH PASSWORD 'mypassword';
CREATE USER myuser WITH PASSWORD 'mypassword' CREATEDB;

-- ユーザー削除
DROP USER myuser;

-- パスワード変更
ALTER USER myuser WITH PASSWORD 'newpassword';

-- 権限付与
GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;
GRANT SELECT, INSERT, UPDATE ON users TO myuser;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO myuser;

-- 権限剥奪
REVOKE ALL PRIVILEGES ON DATABASE mydb FROM myuser;
REVOKE INSERT ON users FROM myuser;

-- ロール作成
CREATE ROLE readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
GRANT readonly TO myuser;
```

## バックアップ・リストア

```bash
# データベース全体をバックアップ
pg_dump -U username -d database > backup.sql
pg_dump -U username -d database -F c -f backup.dump  # カスタム形式

# 特定テーブルのみ
pg_dump -U username -d database -t users > users_backup.sql

# リストア
psql -U username -d database < backup.sql
pg_restore -U username -d database backup.dump

# 全データベースバックアップ
pg_dumpall -U postgres > all_databases.sql
```

## パフォーマンス

### EXPLAIN

```sql
-- 実行計画表示
EXPLAIN SELECT * FROM users WHERE age > 30;

-- 詳細表示
EXPLAIN ANALYZE SELECT * FROM users WHERE age > 30;

-- JSON形式
EXPLAIN (FORMAT JSON, ANALYZE) SELECT * FROM users WHERE age > 30;
```

### VACUUM

```sql
-- テーブルの最適化
VACUUM users;
VACUUM ANALYZE users;  -- 統計情報も更新
VACUUM FULL users;     -- 完全VACUUM（ロック発生）

-- 自動VACUUM設定確認
SHOW autovacuum;
```

### 統計情報

```sql
-- テーブルサイズ
SELECT pg_size_pretty(pg_total_relation_size('users'));

-- データベースサイズ
SELECT pg_size_pretty(pg_database_size('mydb'));

-- 接続数
SELECT COUNT(*) FROM pg_stat_activity;

-- 実行中クエリ
SELECT pid, usename, state, query
FROM pg_stat_activity
WHERE state = 'active';

-- インデックス使用状況
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

## よく使うクエリパターン

```sql
-- ランキング
SELECT
    name,
    age,
    RANK() OVER (ORDER BY age DESC) as rank
FROM users;

-- 行番号
SELECT
    name,
    ROW_NUMBER() OVER (ORDER BY created_at) as row_num
FROM users;

-- ページング（効率的）
SELECT * FROM users
ORDER BY id
LIMIT 20 OFFSET 0;

-- 重複削除（IDを保持）
DELETE FROM users a USING users b
WHERE a.id < b.id AND a.email = b.email;

-- 日付範囲
SELECT * FROM orders
WHERE created_at >= CURRENT_DATE - INTERVAL '7 days';

SELECT * FROM orders
WHERE created_at >= '2024-01-01'
  AND created_at < '2024-02-01';

-- JSONクエリ
SELECT data->>'name' FROM users_json;
SELECT data->'address'->>'city' FROM users_json;
SELECT * FROM users_json WHERE data @> '{"age": 30}';

-- 配列操作
SELECT ARRAY[1, 2, 3];
SELECT * FROM users WHERE tags @> ARRAY['admin'];

-- 文字列連結
SELECT name || ' ' || email FROM users;
SELECT CONCAT(name, ' (', age, ')') FROM users;

-- NULL処理
SELECT COALESCE(phone, 'N/A') FROM users;
SELECT NULLIF(age, 0) FROM users;
```

## Tips

```sql
-- 現在時刻
SELECT NOW();
SELECT CURRENT_TIMESTAMP;
SELECT CURRENT_DATE;
SELECT CURRENT_TIME;

-- 日付操作
SELECT DATE_TRUNC('month', CURRENT_DATE);
SELECT EXTRACT(YEAR FROM CURRENT_DATE);
SELECT AGE(birth_date) FROM users;

-- 文字列操作
SELECT UPPER(name) FROM users;
SELECT LOWER(email) FROM users;
SELECT LENGTH(name) FROM users;
SELECT SUBSTRING(name, 1, 5) FROM users;
SELECT TRIM(name) FROM users;

-- 乱数
SELECT RANDOM();
SELECT FLOOR(RANDOM() * 100);

-- UUID生成
SELECT gen_random_uuid();

-- シーケンス
SELECT nextval('users_id_seq');
SELECT currval('users_id_seq');
SELECT setval('users_id_seq', 100);
```
