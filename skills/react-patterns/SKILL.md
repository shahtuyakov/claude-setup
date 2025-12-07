---
name: react-patterns
description: React and Next.js development patterns for building modern web applications. Use when implementing components, pages, hooks, state management, data fetching, forms, and client-side authentication. Covers TypeScript, App Router, Server Components, and best practices.
---

# React Patterns

Modern React and Next.js patterns with TypeScript.

## Framework Selection

| Setup | Best For | Use When |
|-------|----------|----------|
| Next.js App Router | Full-stack, SSR/SSG | SEO matters, need server components |
| Next.js Pages Router | Simpler SSR | Legacy project, simpler mental model |
| Vite + React | SPA, client-only | No SSR needed, fastest dev experience |
| Create React App | Legacy | Avoid for new projects |

## Reference Files

| Topic | Load | Use When |
|-------|------|----------|
| Project structure | `references/project-structure.md` | Setting up or organizing code |
| Components | `references/components.md` | Building UI components |
| State management | `references/state-management.md` | Managing app/component state |
| Data fetching | `references/data-fetching.md` | API calls, server data |
| Forms | `references/forms.md` | Form handling and validation |
| Auth (client) | `references/auth-client.md` | Client-side authentication |

## Quick Start Patterns

### Next.js App Router Page

```typescript
// app/users/page.tsx
import { getUsers } from '@/lib/api';
import { UserList } from '@/components/users/UserList';

export default async function UsersPage() {
  const users = await getUsers();

  return (
    <main className="container mx-auto py-8">
      <h1 className="text-2xl font-bold mb-6">Users</h1>
      <UserList users={users} />
    </main>
  );
}
```

### Client Component

```typescript
'use client';

import { useState } from 'react';

interface CounterProps {
  initialCount?: number;
}

export function Counter({ initialCount = 0 }: CounterProps) {
  const [count, setCount] = useState(initialCount);

  return (
    <button onClick={() => setCount(c => c + 1)}>
      Count: {count}
    </button>
  );
}
```

### API Route (App Router)

```typescript
// app/api/users/route.ts
import { NextResponse } from 'next/server';

export async function GET() {
  const users = await db.user.findMany();
  return NextResponse.json(users);
}

export async function POST(request: Request) {
  const body = await request.json();
  const user = await db.user.create({ data: body });
  return NextResponse.json(user, { status: 201 });
}
```

## Essential Packages

| Package | Purpose |
|---------|---------|
| `typescript` | Type safety |
| `tailwindcss` | Styling |
| `@tanstack/react-query` | Server state |
| `zustand` | Client state |
| `react-hook-form` | Forms |
| `zod` | Validation |
| `framer-motion` | Animations |
| `lucide-react` | Icons |

## TypeScript Rules

1. **Explicit prop types** - Interface for every component
2. **No `any`** - Use `unknown` if needed
3. **Strict mode** - Enable in tsconfig
4. **Type imports** - Use `import type` when possible

## Code Quality Rules

1. **Functional components** - No class components
2. **Named exports** - Except page defaults
3. **Small files** - One component per file
4. **Custom hooks** - Extract reusable logic
5. **Error boundaries** - Wrap key sections
6. **Loading states** - Always handle loading/error
