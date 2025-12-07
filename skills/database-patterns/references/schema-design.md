# Schema Design

## Normalization Levels

| Form | Rule | Example |
|------|------|---------|
| 1NF | Atomic values, no repeating groups | Split `tags: "a,b,c"` into rows |
| 2NF | 1NF + no partial dependencies | Separate order items from orders |
| 3NF | 2NF + no transitive dependencies | Separate city from zip code |

**Recommendation**: Design to 3NF, then denormalize strategically for performance.

## Standard Table Template

```sql
CREATE TABLE table_name (
    -- Primary key
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Business columns
    -- ...

    -- Timestamps
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Soft delete
    deleted_at TIMESTAMPTZ
);

-- Updated_at trigger
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER table_name_updated_at
    BEFORE UPDATE ON table_name
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();
```

## Primary Key Strategies

| Type | Pros | Cons | Use When |
|------|------|------|----------|
| UUID v4 | No collisions, distributed | Larger (16 bytes), random | Distributed systems |
| UUID v7 | Sortable, distributed | Larger (16 bytes) | Need sorting + distributed |
| SERIAL/BIGSERIAL | Compact, fast, sortable | Single point of failure | Single database |
| ULID | Sortable, compact string | Less common | Need string IDs + sorting |

```sql
-- UUID v4 (random)
id UUID PRIMARY KEY DEFAULT gen_random_uuid()

-- UUID v7 (time-sortable) - requires extension
CREATE EXTENSION IF NOT EXISTS pg_uuidv7;
id UUID PRIMARY KEY DEFAULT uuid_generate_v7()

-- SERIAL
id SERIAL PRIMARY KEY

-- BIGSERIAL (for large tables)
id BIGSERIAL PRIMARY KEY
```

## Relationships

### One-to-Many

```sql
-- Parent
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL
);

-- Child (many)
CREATE TABLE posts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    content TEXT
);

CREATE INDEX posts_user_id_idx ON posts(user_id);
```

### Many-to-Many

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL
);

CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(50) NOT NULL UNIQUE
);

-- Junction table
CREATE TABLE user_roles (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    granted_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    granted_by UUID REFERENCES users(id),
    PRIMARY KEY (user_id, role_id)
);

CREATE INDEX user_roles_role_id_idx ON user_roles(role_id);
```

### Self-Referential

```sql
-- Comments with replies
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id UUID REFERENCES comments(id) ON DELETE CASCADE,
    content TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX comments_parent_id_idx ON comments(parent_id);

-- Categories with hierarchy
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    name VARCHAR(100) NOT NULL,
    path LTREE -- For efficient tree queries (requires ltree extension)
);
```

## Common Patterns

### Soft Delete

```sql
-- Table with soft delete
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL,
    deleted_at TIMESTAMPTZ,
    -- Unique constraint only for non-deleted
    CONSTRAINT users_email_unique UNIQUE (email) WHERE deleted_at IS NULL
);

-- Partial index for active records
CREATE INDEX users_active_idx ON users(email) WHERE deleted_at IS NULL;

-- Query pattern
SELECT * FROM users WHERE deleted_at IS NULL;
```

### Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_name VARCHAR(100) NOT NULL,
    record_id UUID NOT NULL,
    action VARCHAR(10) NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
    old_data JSONB,
    new_data JSONB,
    changed_by UUID,
    changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX audit_log_table_record_idx ON audit_log(table_name, record_id);
CREATE INDEX audit_log_changed_at_idx ON audit_log(changed_at);
```

### Polymorphic Association

```sql
-- Option 1: Type column
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    commentable_type VARCHAR(50) NOT NULL, -- 'post', 'video', etc.
    commentable_id UUID NOT NULL,
    content TEXT NOT NULL
);

CREATE INDEX comments_commentable_idx ON comments(commentable_type, commentable_id);

-- Option 2: Multiple foreign keys (preferred for type safety)
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    post_id UUID REFERENCES posts(id) ON DELETE CASCADE,
    video_id UUID REFERENCES videos(id) ON DELETE CASCADE,
    content TEXT NOT NULL,
    CONSTRAINT comments_single_parent CHECK (
        (post_id IS NOT NULL)::int + (video_id IS NOT NULL)::int = 1
    )
);
```

### Status/State Machine

```sql
CREATE TYPE order_status AS ENUM (
    'pending',
    'confirmed',
    'processing',
    'shipped',
    'delivered',
    'cancelled'
);

CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    status order_status NOT NULL DEFAULT 'pending',
    status_changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Track status history
CREATE TABLE order_status_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    from_status order_status,
    to_status order_status NOT NULL,
    changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    changed_by UUID REFERENCES users(id)
);
```

### Tags/Labels

```sql
-- Normalized approach
CREATE TABLE tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE post_tags (
    post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    tag_id UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (post_id, tag_id)
);

-- Array approach (simpler, less flexible)
CREATE TABLE posts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(255) NOT NULL,
    tags TEXT[] NOT NULL DEFAULT '{}'
);

CREATE INDEX posts_tags_idx ON posts USING GIN(tags);

-- Query: SELECT * FROM posts WHERE 'typescript' = ANY(tags);
```

### Versioning/History

```sql
-- Current table
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    version INT NOT NULL DEFAULT 1,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- History table
CREATE TABLE document_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    version INT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID REFERENCES users(id)
);

CREATE UNIQUE INDEX document_versions_doc_version_idx
    ON document_versions(document_id, version);
```

## Data Types

| Use Case | PostgreSQL Type |
|----------|-----------------|
| Text (short) | VARCHAR(n) |
| Text (long) | TEXT |
| Numbers | INTEGER, BIGINT, NUMERIC(p,s) |
| Money | NUMERIC(19,4) or INTEGER (cents) |
| Boolean | BOOLEAN |
| Date only | DATE |
| Date + time | TIMESTAMPTZ |
| Duration | INTERVAL |
| JSON | JSONB |
| Arrays | type[] |
| IP Address | INET |
| UUID | UUID |

## Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Tables | snake_case, singular | `user`, `order_item` |
| Columns | snake_case | `created_at`, `user_id` |
| Primary key | `id` | `id UUID` |
| Foreign key | `{table}_id` | `user_id`, `post_id` |
| Indexes | `{table}_{columns}_idx` | `users_email_idx` |
| Unique | `{table}_{columns}_unique` | `users_email_unique` |
| Check | `{table}_{column}_check` | `orders_amount_check` |

## Anti-Patterns to Avoid

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| God table | Too many columns | Split into related tables |
| EAV pattern | Poor performance, no typing | Use JSONB or proper columns |
| Stringly typed | No validation | Use ENUM or reference table |
| No constraints | Data integrity issues | Add FK, CHECK, UNIQUE |
| No indexes | Slow queries | Index filtered/joined columns |
