# Data Fetching

## When to Use What

| Method | Use When |
|--------|----------|
| Server Component fetch | Static/SSR data, SEO content |
| React Query | Client-side, caching, mutations |
| SWR | Simpler client-side caching |
| Server Actions | Form mutations, revalidation |

## Server Component Fetching (App Router)

### Basic Fetch

```typescript
// app/users/page.tsx
async function getUsers() {
  const res = await fetch('https://api.example.com/users', {
    next: { revalidate: 3600 }, // Revalidate every hour
  });
  if (!res.ok) throw new Error('Failed to fetch');
  return res.json();
}

export default async function UsersPage() {
  const users = await getUsers();

  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

### Caching Options

```typescript
// Static (default) - cached until redeployed
fetch(url);

// Revalidate every 60 seconds
fetch(url, { next: { revalidate: 60 } });

// No cache - always fresh
fetch(url, { cache: 'no-store' });

// Revalidate on demand
fetch(url, { next: { tags: ['users'] } });
// Then call: revalidateTag('users')
```

### Parallel Fetching

```typescript
export default async function Dashboard() {
  // Parallel - faster
  const [users, posts, stats] = await Promise.all([
    getUsers(),
    getPosts(),
    getStats(),
  ]);

  return (
    <div>
      <UserList users={users} />
      <PostList posts={posts} />
      <StatsCard stats={stats} />
    </div>
  );
}
```

### Loading and Error States

```typescript
// app/users/loading.tsx
export default function Loading() {
  return <div className="animate-pulse">Loading users...</div>;
}

// app/users/error.tsx
'use client';

export default function Error({
  error,
  reset,
}: {
  error: Error;
  reset: () => void;
}) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

## React Query (Client-Side)

### Setup

```bash
npm install @tanstack/react-query
```

### Provider Setup

```typescript
// providers/QueryProvider.tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useState } from 'react';

export function QueryProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000, // 1 minute
            refetchOnWindowFocus: false,
          },
        },
      })
  );

  return (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
}

// app/layout.tsx
import { QueryProvider } from '@/providers/QueryProvider';

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <QueryProvider>{children}</QueryProvider>
      </body>
    </html>
  );
}
```

### Basic Query

```typescript
'use client';

import { useQuery } from '@tanstack/react-query';

async function fetchUsers() {
  const res = await fetch('/api/users');
  if (!res.ok) throw new Error('Failed to fetch');
  return res.json();
}

export function UserList() {
  const { data, isLoading, error } = useQuery({
    queryKey: ['users'],
    queryFn: fetchUsers,
  });

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <ul>
      {data.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

### Query with Parameters

```typescript
function useUser(userId: string) {
  return useQuery({
    queryKey: ['users', userId],
    queryFn: () => fetch(`/api/users/${userId}`).then((r) => r.json()),
    enabled: !!userId, // Only fetch if userId exists
  });
}

// Usage
const { data: user } = useUser(userId);
```

### Mutations

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';

export function CreateUserForm() {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: (newUser: { name: string; email: string }) =>
      fetch('/api/users', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(newUser),
      }).then((r) => r.json()),
    onSuccess: () => {
      // Invalidate and refetch
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    mutation.mutate({
      name: formData.get('name') as string,
      email: formData.get('email') as string,
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" required />
      <input name="email" type="email" required />
      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? 'Creating...' : 'Create User'}
      </button>
      {mutation.error && <p>Error: {mutation.error.message}</p>}
    </form>
  );
}
```

### Optimistic Updates

```typescript
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    // Cancel outgoing queries
    await queryClient.cancelQueries({ queryKey: ['todos'] });

    // Snapshot previous value
    const previousTodos = queryClient.getQueryData(['todos']);

    // Optimistically update
    queryClient.setQueryData(['todos'], (old) =>
      old.map((todo) => (todo.id === newTodo.id ? newTodo : todo))
    );

    return { previousTodos };
  },
  onError: (err, newTodo, context) => {
    // Rollback on error
    queryClient.setQueryData(['todos'], context.previousTodos);
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] });
  },
});
```

### Infinite Queries (Pagination)

```typescript
import { useInfiniteQuery } from '@tanstack/react-query';

function useInfinitePosts() {
  return useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: ({ pageParam = 1 }) =>
      fetch(`/api/posts?page=${pageParam}`).then((r) => r.json()),
    getNextPageParam: (lastPage) => lastPage.nextPage,
  });
}

export function PostsList() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } =
    useInfinitePosts();

  return (
    <div>
      {data?.pages.map((page) =>
        page.posts.map((post) => <PostCard key={post.id} post={post} />)
      )}
      <button
        onClick={() => fetchNextPage()}
        disabled={!hasNextPage || isFetchingNextPage}
      >
        {isFetchingNextPage ? 'Loading...' : hasNextPage ? 'Load More' : 'Done'}
      </button>
    </div>
  );
}
```

## Server Actions (App Router)

### Basic Action

```typescript
// app/actions.ts
'use server';

import { revalidatePath } from 'next/cache';

export async function createUser(formData: FormData) {
  const name = formData.get('name');
  const email = formData.get('email');

  await db.user.create({ data: { name, email } });

  revalidatePath('/users');
}

// Usage in component
import { createUser } from './actions';

export function CreateUserForm() {
  return (
    <form action={createUser}>
      <input name="name" required />
      <input name="email" type="email" required />
      <button type="submit">Create</button>
    </form>
  );
}
```

### With useFormState

```typescript
'use client';

import { useFormState, useFormStatus } from 'react-dom';
import { createUser } from './actions';

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Creating...' : 'Create'}
    </button>
  );
}

export function CreateUserForm() {
  const [state, formAction] = useFormState(createUser, { error: null });

  return (
    <form action={formAction}>
      <input name="name" required />
      <input name="email" type="email" required />
      <SubmitButton />
      {state.error && <p className="text-red-500">{state.error}</p>}
    </form>
  );
}
```

## API Client Pattern

```typescript
// lib/api.ts
const BASE_URL = process.env.NEXT_PUBLIC_API_URL || '/api';

async function fetcher<T>(
  endpoint: string,
  options?: RequestInit
): Promise<T> {
  const res = await fetch(`${BASE_URL}${endpoint}`, {
    headers: {
      'Content-Type': 'application/json',
      ...options?.headers,
    },
    ...options,
  });

  if (!res.ok) {
    const error = await res.json().catch(() => ({}));
    throw new Error(error.message || 'API Error');
  }

  return res.json();
}

export const api = {
  users: {
    list: () => fetcher<User[]>('/users'),
    get: (id: string) => fetcher<User>(`/users/${id}`),
    create: (data: CreateUserDto) =>
      fetcher<User>('/users', {
        method: 'POST',
        body: JSON.stringify(data),
      }),
    update: (id: string, data: UpdateUserDto) =>
      fetcher<User>(`/users/${id}`, {
        method: 'PATCH',
        body: JSON.stringify(data),
      }),
    delete: (id: string) =>
      fetcher<void>(`/users/${id}`, { method: 'DELETE' }),
  },
};

// Usage with React Query
const { data } = useQuery({
  queryKey: ['users'],
  queryFn: api.users.list,
});
```

## Best Practices

1. **Server Components by default** - Only use client-side for interactivity
2. **Parallel fetching** - Use Promise.all for independent requests
3. **Cache appropriately** - Set staleTime based on data freshness needs
4. **Handle all states** - Loading, error, empty, success
5. **Invalidate on mutation** - Keep cache in sync
6. **Use query keys wisely** - Include all dependencies
