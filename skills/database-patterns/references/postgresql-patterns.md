# PostgreSQL Patterns

## Index Types

| Type | Use For | Example |
|------|---------|---------|
| B-tree | Equality, range, sorting | `CREATE INDEX idx ON t(col)` |
| Hash | Equality only | `CREATE INDEX idx ON t USING HASH(col)` |
| GIN | Arrays, JSONB, full-text | `CREATE INDEX idx ON t USING GIN(col)` |
| GiST | Geometric, range types | `CREATE INDEX idx ON t USING GIST(col)` |
| BRIN | Large sorted tables | `CREATE INDEX idx ON t USING BRIN(col)` |

## Index Patterns

### Basic Index

```sql
-- Single column
CREATE INDEX users_email_idx ON users(email);

-- Composite (order matters!)
CREATE INDEX posts_user_created_idx ON posts(user_id, created_at DESC);

-- Unique
CREATE UNIQUE INDEX users_email_unique ON users(email);
```

### Partial Index

```sql
-- Only index active records
CREATE INDEX users_active_email_idx ON users(email)
    WHERE deleted_at IS NULL;

-- Only index pending orders
CREATE INDEX orders_pending_idx ON orders(created_at)
    WHERE status = 'pending';
```

### Expression Index

```sql
-- Index lowercase email
CREATE INDEX users_email_lower_idx ON users(LOWER(email));

-- Index JSONB field
CREATE INDEX users_settings_theme_idx ON users((settings->>'theme'));

-- Index date part
CREATE INDEX orders_created_date_idx ON orders(DATE(created_at));
```

### Covering Index

```sql
-- Include columns in index (avoid table lookup)
CREATE INDEX posts_user_id_idx ON posts(user_id)
    INCLUDE (title, created_at);

-- Query uses index-only scan
SELECT title, created_at FROM posts WHERE user_id = ?;
```

## JSONB Operations

### Storage and Queries

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY,
    settings JSONB NOT NULL DEFAULT '{}'
);

-- Insert
INSERT INTO users (id, settings)
VALUES (gen_random_uuid(), '{"theme": "dark", "notifications": true}');

-- Query
SELECT * FROM users WHERE settings->>'theme' = 'dark';
SELECT * FROM users WHERE settings @> '{"theme": "dark"}';
SELECT * FROM users WHERE settings ? 'notifications';

-- Update
UPDATE users SET settings = settings || '{"language": "en"}';
UPDATE users SET settings = settings - 'notifications';
UPDATE users SET settings = jsonb_set(settings, '{theme}', '"light"');
```

### JSONB Indexing

```sql
-- GIN index for @>, ?, ?|, ?& operators
CREATE INDEX users_settings_idx ON users USING GIN(settings);

-- Path-specific index
CREATE INDEX users_settings_theme_idx ON users((settings->>'theme'));
```

## Full-Text Search

### Basic Setup

```sql
-- Add search vector column
ALTER TABLE posts ADD COLUMN search_vector tsvector;

-- Populate
UPDATE posts SET search_vector =
    setweight(to_tsvector('english', title), 'A') ||
    setweight(to_tsvector('english', content), 'B');

-- GIN index
CREATE INDEX posts_search_idx ON posts USING GIN(search_vector);

-- Search
SELECT * FROM posts
WHERE search_vector @@ plainto_tsquery('english', 'database optimization');
```

### Auto-Update Trigger

```sql
CREATE OR REPLACE FUNCTION posts_search_trigger()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(NEW.content, '')), 'B');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER posts_search_update
    BEFORE INSERT OR UPDATE ON posts
    FOR EACH ROW
    EXECUTE FUNCTION posts_search_trigger();
```

## Common Table Expressions (CTEs)

### Recursive CTE (Hierarchies)

```sql
-- Get all descendants of a category
WITH RECURSIVE category_tree AS (
    -- Base case
    SELECT id, name, parent_id, 0 AS depth
    FROM categories
    WHERE id = '123'

    UNION ALL

    -- Recursive case
    SELECT c.id, c.name, c.parent_id, ct.depth + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree;
```

### CTE for Readability

```sql
WITH active_users AS (
    SELECT * FROM users WHERE deleted_at IS NULL
),
recent_posts AS (
    SELECT * FROM posts WHERE created_at > NOW() - INTERVAL '7 days'
)
SELECT u.name, COUNT(p.id) AS post_count
FROM active_users u
LEFT JOIN recent_posts p ON p.user_id = u.id
GROUP BY u.id, u.name;
```

## Window Functions

```sql
-- Row number
SELECT *,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
FROM posts;

-- Running total
SELECT *,
    SUM(amount) OVER (ORDER BY created_at) AS running_total
FROM transactions;

-- Rank
SELECT *,
    RANK() OVER (PARTITION BY category_id ORDER BY score DESC) AS rank
FROM products;

-- Lag/Lead (previous/next row)
SELECT *,
    LAG(price) OVER (ORDER BY date) AS previous_price,
    LEAD(price) OVER (ORDER BY date) AS next_price
FROM price_history;
```

## Pagination

### Offset Pagination (Simple, Slow for Large Offsets)

```sql
SELECT * FROM posts
ORDER BY created_at DESC
LIMIT 20 OFFSET 40;
```

### Cursor Pagination (Efficient)

```sql
-- First page
SELECT * FROM posts
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Next page (using last item's values)
SELECT * FROM posts
WHERE (created_at, id) < ('2024-01-01 12:00:00', 'last-uuid')
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

### Keyset Pagination with Total

```sql
WITH data AS (
    SELECT * FROM posts
    WHERE (created_at, id) < ($1, $2) OR $1 IS NULL
    ORDER BY created_at DESC, id DESC
    LIMIT $3
),
total AS (
    SELECT COUNT(*) FROM posts
)
SELECT d.*, t.count AS total_count
FROM data d, total t;
```

## Upsert (INSERT ON CONFLICT)

```sql
-- Insert or update
INSERT INTO users (email, name, updated_at)
VALUES ('john@example.com', 'John', NOW())
ON CONFLICT (email)
DO UPDATE SET
    name = EXCLUDED.name,
    updated_at = EXCLUDED.updated_at;

-- Insert or ignore
INSERT INTO tags (name)
VALUES ('typescript')
ON CONFLICT (name) DO NOTHING;

-- Bulk upsert
INSERT INTO products (sku, name, price)
VALUES
    ('SKU1', 'Product 1', 100),
    ('SKU2', 'Product 2', 200)
ON CONFLICT (sku)
DO UPDATE SET
    name = EXCLUDED.name,
    price = EXCLUDED.price;
```

## Transactions

```sql
BEGIN;

-- Savepoint for partial rollback
SAVEPOINT before_update;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- Check constraint
SELECT balance FROM accounts WHERE id = 1;
-- If negative, rollback to savepoint
ROLLBACK TO SAVEPOINT before_update;

COMMIT;
```

## Query Optimization

### EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE
SELECT u.name, COUNT(p.id)
FROM users u
LEFT JOIN posts p ON p.user_id = u.id
WHERE u.deleted_at IS NULL
GROUP BY u.id;

-- Look for:
-- - Seq Scan (may need index)
-- - Nested Loop (may need index on join column)
-- - Sort (may need index for ORDER BY)
-- - High "actual time"
```

### Common Optimizations

```sql
-- Avoid SELECT *
SELECT id, name, email FROM users;  -- Not SELECT *

-- Use EXISTS instead of COUNT for existence check
SELECT EXISTS(SELECT 1 FROM users WHERE email = 'test@example.com');

-- Use IN for small sets, EXISTS for subqueries
SELECT * FROM users WHERE id IN (1, 2, 3);
SELECT * FROM users u WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);

-- Batch inserts
INSERT INTO logs (message) VALUES
    ('log 1'),
    ('log 2'),
    ('log 3');
```

## Useful Extensions

| Extension | Purpose |
|-----------|---------|
| `uuid-ossp` | UUID generation |
| `pg_trgm` | Trigram similarity (fuzzy search) |
| `pgcrypto` | Encryption functions |
| `pg_stat_statements` | Query statistics |
| `postgis` | Geographic data |
| `pgvector` | Vector embeddings (AI) |
| `ltree` | Hierarchical labels |
| `pg_uuidv7` | Time-sortable UUIDs |

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Fuzzy search with trigram
CREATE INDEX users_name_trgm_idx ON users USING GIN(name gin_trgm_ops);
SELECT * FROM users WHERE name % 'john';  -- Similarity search
```
