# Database Migrations

## Migration Principles

1. **One migration = one logical change** - Atomic, focused
2. **Immutable after deployment** - Never edit deployed migrations
3. **Backward compatible** - Old code works with new schema
4. **Include rollback** - Down migration for every up
5. **Test in staging first** - Before production
6. **Descriptive names** - `add_user_email_verification`

## Drizzle Migrations

### Setup

```bash
# Generate migration
npx drizzle-kit generate

# Apply migrations
npx drizzle-kit migrate

# Push (dev only - no migration files)
npx drizzle-kit push
```

### Generated Migration Structure

```
drizzle/
├── 0000_initial.sql
├── 0001_add_posts.sql
├── 0002_add_user_status.sql
└── meta/
    ├── _journal.json
    └── 0000_snapshot.json
```

### Example Migrations

```sql
-- 0001_add_posts.sql
CREATE TABLE IF NOT EXISTS "posts" (
    "id" uuid PRIMARY KEY DEFAULT gen_random_uuid() NOT NULL,
    "user_id" uuid NOT NULL,
    "title" varchar(255) NOT NULL,
    "content" text,
    "created_at" timestamp with time zone DEFAULT now() NOT NULL,
    CONSTRAINT "posts_user_id_users_id_fk" FOREIGN KEY ("user_id")
        REFERENCES "users"("id") ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS "posts_user_id_idx" ON "posts"("user_id");
```

### Custom Migration

```typescript
// src/db/migrate.ts
import { drizzle } from 'drizzle-orm/postgres-js';
import { migrate } from 'drizzle-orm/postgres-js/migrator';
import postgres from 'postgres';

const runMigrations = async () => {
  const connection = postgres(process.env.DATABASE_URL!, { max: 1 });
  const db = drizzle(connection);

  console.log('Running migrations...');
  await migrate(db, { migrationsFolder: './drizzle' });
  console.log('Migrations complete');

  await connection.end();
};

runMigrations();
```

---

## Prisma Migrations

### Commands

```bash
# Create migration
npx prisma migrate dev --name add_posts

# Apply migrations (production)
npx prisma migrate deploy

# Reset database (dev only)
npx prisma migrate reset

# View migration status
npx prisma migrate status
```

### Migration Structure

```
prisma/
├── schema.prisma
└── migrations/
    ├── 20240101000000_initial/
    │   └── migration.sql
    ├── 20240102000000_add_posts/
    │   └── migration.sql
    └── migration_lock.toml
```

### Custom Migration SQL

```sql
-- prisma/migrations/20240103000000_add_user_status/migration.sql

-- Create enum type
CREATE TYPE "UserStatus" AS ENUM ('active', 'inactive', 'pending');

-- Add column with default
ALTER TABLE "users" ADD COLUMN "status" "UserStatus" NOT NULL DEFAULT 'pending';

-- Backfill existing data
UPDATE "users" SET "status" = 'active' WHERE "email_verified_at" IS NOT NULL;
```

---

## Backward Compatible Migrations

### Adding a Column

```sql
-- GOOD: Add nullable or with default
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
ALTER TABLE users ADD COLUMN status VARCHAR(20) NOT NULL DEFAULT 'active';

-- BAD: Add NOT NULL without default (blocks writes)
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NOT NULL;
```

### Removing a Column

```sql
-- Step 1: Stop using column in code (deploy)
-- Step 2: Wait for all instances to update
-- Step 3: Drop column

ALTER TABLE users DROP COLUMN old_column;
```

### Renaming a Column

```sql
-- Step 1: Add new column
ALTER TABLE users ADD COLUMN new_name VARCHAR(100);

-- Step 2: Backfill data
UPDATE users SET new_name = old_name;

-- Step 3: Add NOT NULL constraint
ALTER TABLE users ALTER COLUMN new_name SET NOT NULL;

-- Step 4: Update code to use new column (deploy)

-- Step 5: Drop old column
ALTER TABLE users DROP COLUMN old_name;
```

### Changing Column Type

```sql
-- Step 1: Add new column with new type
ALTER TABLE orders ADD COLUMN amount_new NUMERIC(19,4);

-- Step 2: Backfill
UPDATE orders SET amount_new = amount::NUMERIC(19,4);

-- Step 3: Swap in code

-- Step 4: Drop old, rename new
ALTER TABLE orders DROP COLUMN amount;
ALTER TABLE orders RENAME COLUMN amount_new TO amount;
```

### Adding an Index

```sql
-- GOOD: Concurrent (doesn't lock table)
CREATE INDEX CONCURRENTLY users_email_idx ON users(email);

-- BAD: Regular (locks table)
CREATE INDEX users_email_idx ON users(email);
```

---

## Migration Patterns

### Data Migration with SQL

```sql
-- Insert reference data
INSERT INTO roles (id, name) VALUES
    (gen_random_uuid(), 'admin'),
    (gen_random_uuid(), 'user'),
    (gen_random_uuid(), 'guest')
ON CONFLICT (name) DO NOTHING;

-- Transform existing data
UPDATE users
SET status = CASE
    WHEN email_verified_at IS NOT NULL THEN 'active'
    ELSE 'pending'
END;
```

### Split Migration (Large Tables)

```sql
-- For large tables, split into batches
DO $$
DECLARE
    batch_size INT := 10000;
    affected INT := 1;
BEGIN
    WHILE affected > 0 LOOP
        UPDATE users
        SET status = 'active'
        WHERE id IN (
            SELECT id FROM users
            WHERE status IS NULL
            LIMIT batch_size
        );
        GET DIAGNOSTICS affected = ROW_COUNT;
        COMMIT;
    END LOOP;
END $$;
```

### Rollback Migration

```sql
-- Up: Add column
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Down: Remove column
ALTER TABLE users DROP COLUMN phone;
```

---

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/migrate.yml
name: Database Migration

on:
  push:
    branches: [main]
    paths:
      - 'prisma/migrations/**'
      - 'drizzle/**'

jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run migrations
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

### Pre-deployment Checks

```typescript
// scripts/check-migrations.ts
import { execSync } from 'child_process';

// Check for pending migrations
const status = execSync('npx prisma migrate status', {
  encoding: 'utf8',
});

if (status.includes('not yet applied')) {
  console.error('Pending migrations detected!');
  process.exit(1);
}

console.log('All migrations applied');
```

---

## Troubleshooting

### Migration Failed

```bash
# Check status
npx prisma migrate status

# Mark migration as applied (if manually fixed)
npx prisma migrate resolve --applied "20240101000000_migration_name"

# Mark as rolled back
npx prisma migrate resolve --rolled-back "20240101000000_migration_name"
```

### Schema Drift

```bash
# Check for drift
npx prisma migrate diff \
  --from-schema-datasource prisma/schema.prisma \
  --to-schema-datamodel prisma/schema.prisma

# Create migration for drift
npx prisma migrate dev --create-only
```

### Rollback in Production

```sql
-- Manual rollback (have scripts ready)
-- 1. Identify failing migration
-- 2. Run corresponding down.sql
-- 3. Update migration tracking table
DELETE FROM _prisma_migrations
WHERE migration_name = '20240101000000_failing_migration';
```

## Best Practices Checklist

- [ ] Migration files are in version control
- [ ] Each migration has a descriptive name
- [ ] Backward compatibility verified
- [ ] Tested on staging environment
- [ ] Rollback script prepared
- [ ] Large table migrations use batching
- [ ] Indexes created concurrently
- [ ] Data backups taken before deployment
