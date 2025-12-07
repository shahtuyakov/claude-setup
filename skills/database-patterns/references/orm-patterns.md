# ORM Patterns

## Drizzle ORM (Recommended for Performance)

### Setup

```bash
npm install drizzle-orm postgres
npm install -D drizzle-kit
```

### Configuration

```typescript
// drizzle.config.ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  schema: './src/db/schema.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
});
```

### Schema Definition

```typescript
// src/db/schema.ts
import {
  pgTable,
  uuid,
  varchar,
  text,
  timestamp,
  integer,
  boolean,
  pgEnum,
} from 'drizzle-orm/pg-core';
import { relations } from 'drizzle-orm';

// Enums
export const userStatusEnum = pgEnum('user_status', ['active', 'inactive', 'pending']);

// Users table
export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: varchar('email', { length: 255 }).notNull().unique(),
  name: varchar('name', { length: 100 }).notNull(),
  passwordHash: varchar('password_hash', { length: 255 }).notNull(),
  status: userStatusEnum('status').notNull().default('pending'),
  emailVerifiedAt: timestamp('email_verified_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
  deletedAt: timestamp('deleted_at', { withTimezone: true }),
});

// Posts table
export const posts = pgTable('posts', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  title: varchar('title', { length: 255 }).notNull(),
  content: text('content'),
  published: boolean('published').notNull().default(false),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// Relations
export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}));

export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, {
    fields: [posts.userId],
    references: [users.id],
  }),
}));

// Types
export type User = typeof users.$inferSelect;
export type NewUser = typeof users.$inferInsert;
export type Post = typeof posts.$inferSelect;
export type NewPost = typeof posts.$inferInsert;
```

### Database Connection

```typescript
// src/db/index.ts
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';
import * as schema from './schema';

const connectionString = process.env.DATABASE_URL!;
const client = postgres(connectionString);

export const db = drizzle(client, { schema });
```

### CRUD Operations

```typescript
import { db } from './db';
import { users, posts } from './db/schema';
import { eq, and, desc, like, sql } from 'drizzle-orm';

// Create
const newUser = await db.insert(users).values({
  email: 'john@example.com',
  name: 'John Doe',
  passwordHash: hashedPassword,
}).returning();

// Read - single
const user = await db.query.users.findFirst({
  where: eq(users.email, 'john@example.com'),
});

// Read - with relations
const userWithPosts = await db.query.users.findFirst({
  where: eq(users.id, userId),
  with: {
    posts: {
      orderBy: [desc(posts.createdAt)],
      limit: 10,
    },
  },
});

// Read - list with filters
const activeUsers = await db
  .select()
  .from(users)
  .where(and(
    eq(users.status, 'active'),
    like(users.name, '%john%')
  ))
  .orderBy(desc(users.createdAt))
  .limit(20);

// Update
await db.update(users)
  .set({ name: 'Jane Doe', updatedAt: new Date() })
  .where(eq(users.id, userId));

// Delete
await db.delete(users).where(eq(users.id, userId));

// Soft delete
await db.update(users)
  .set({ deletedAt: new Date() })
  .where(eq(users.id, userId));

// Upsert
await db.insert(users)
  .values({ email, name, passwordHash })
  .onConflictDoUpdate({
    target: users.email,
    set: { name, updatedAt: new Date() },
  });
```

### Transactions

```typescript
await db.transaction(async (tx) => {
  const [user] = await tx.insert(users).values({
    email: 'john@example.com',
    name: 'John',
    passwordHash,
  }).returning();

  await tx.insert(posts).values({
    userId: user.id,
    title: 'First Post',
    content: 'Hello World',
  });
});
```

### Raw SQL

```typescript
// Prepared statement
const result = await db.execute(
  sql`SELECT * FROM users WHERE email = ${email}`
);

// Complex query
const stats = await db.execute(sql`
  SELECT
    DATE(created_at) as date,
    COUNT(*) as count
  FROM posts
  WHERE created_at > NOW() - INTERVAL '30 days'
  GROUP BY DATE(created_at)
  ORDER BY date DESC
`);
```

---

## Prisma ORM (Recommended for DX)

### Setup

```bash
npm install prisma @prisma/client
npx prisma init
```

### Schema Definition

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum UserStatus {
  active
  inactive
  pending
}

model User {
  id              String      @id @default(uuid())
  email           String      @unique
  name            String
  passwordHash    String      @map("password_hash")
  status          UserStatus  @default(pending)
  emailVerifiedAt DateTime?   @map("email_verified_at")
  createdAt       DateTime    @default(now()) @map("created_at")
  updatedAt       DateTime    @updatedAt @map("updated_at")
  deletedAt       DateTime?   @map("deleted_at")

  posts           Post[]

  @@map("users")
}

model Post {
  id        String   @id @default(uuid())
  userId    String   @map("user_id")
  title     String
  content   String?
  published Boolean  @default(false)
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  author    User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@map("posts")
}
```

### CRUD Operations

```typescript
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

// Create
const user = await prisma.user.create({
  data: {
    email: 'john@example.com',
    name: 'John Doe',
    passwordHash,
  },
});

// Create with relation
const userWithPost = await prisma.user.create({
  data: {
    email: 'john@example.com',
    name: 'John',
    passwordHash,
    posts: {
      create: {
        title: 'First Post',
        content: 'Hello World',
      },
    },
  },
  include: { posts: true },
});

// Read - single
const user = await prisma.user.findUnique({
  where: { email: 'john@example.com' },
});

// Read - with relations
const userWithPosts = await prisma.user.findUnique({
  where: { id: userId },
  include: {
    posts: {
      orderBy: { createdAt: 'desc' },
      take: 10,
    },
  },
});

// Read - list with filters
const activeUsers = await prisma.user.findMany({
  where: {
    status: 'active',
    name: { contains: 'john', mode: 'insensitive' },
    deletedAt: null,
  },
  orderBy: { createdAt: 'desc' },
  take: 20,
});

// Update
await prisma.user.update({
  where: { id: userId },
  data: { name: 'Jane Doe' },
});

// Upsert
await prisma.user.upsert({
  where: { email },
  update: { name },
  create: { email, name, passwordHash },
});

// Delete
await prisma.user.delete({
  where: { id: userId },
});

// Soft delete middleware
prisma.$use(async (params, next) => {
  if (params.model === 'User' && params.action === 'delete') {
    params.action = 'update';
    params.args.data = { deletedAt: new Date() };
  }
  return next(params);
});
```

### Transactions

```typescript
// Interactive transaction
const [user, post] = await prisma.$transaction(async (tx) => {
  const user = await tx.user.create({
    data: { email, name, passwordHash },
  });

  const post = await tx.post.create({
    data: { userId: user.id, title: 'First Post' },
  });

  return [user, post];
});

// Batch transaction
await prisma.$transaction([
  prisma.user.create({ data: userData }),
  prisma.post.create({ data: postData }),
]);
```

### Raw SQL

```typescript
// Raw query
const users = await prisma.$queryRaw`
  SELECT * FROM users WHERE email = ${email}
`;

// Raw execute
await prisma.$executeRaw`
  UPDATE users SET status = 'active' WHERE id = ${id}
`;
```

---

## Comparison

| Feature | Drizzle | Prisma |
|---------|---------|--------|
| Bundle size | ~7kb | ~2MB+ |
| Query syntax | SQL-like | Object-based |
| Type inference | Runtime | Codegen |
| Serverless | Excellent | Slower cold start |
| Learning curve | Know SQL | Abstracted |
| Migrations | drizzle-kit | prisma migrate |
| Raw SQL | Easy | $queryRaw |
| Relations | Manual setup | Declarative |

## Best Practices

1. **Use transactions** - For multi-table operations
2. **Select only needed columns** - Better performance
3. **Index foreign keys** - Always
4. **Use prepared statements** - Prevents SQL injection
5. **Handle soft deletes** - Add middleware/filter
6. **Paginate large results** - Never load all
7. **Use connection pooling** - For serverless
