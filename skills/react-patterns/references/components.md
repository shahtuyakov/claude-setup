# Component Patterns

## Basic Component

```typescript
interface CardProps {
  title: string;
  children: React.ReactNode;
  className?: string;
}

export function Card({ title, children, className }: CardProps) {
  return (
    <div className={cn('rounded-lg border p-4', className)}>
      <h3 className="font-semibold">{title}</h3>
      {children}
    </div>
  );
}
```

## Server vs Client Components

### Server Component (Default in App Router)

```typescript
// app/users/page.tsx
// No 'use client' - runs on server
import { getUsers } from '@/lib/api';

export default async function UsersPage() {
  const users = await getUsers(); // Direct DB/API call

  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

### Client Component

```typescript
'use client';

import { useState } from 'react';

export function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(c => c + 1)}>
      Count: {count}
    </button>
  );
}
```

### When to Use Which

| Server Component | Client Component |
|------------------|------------------|
| Fetch data | useState, useEffect |
| Access backend resources | Event handlers (onClick) |
| Keep sensitive info server-side | Browser APIs |
| Large dependencies | Interactivity |

## Props Patterns

### Required vs Optional

```typescript
interface UserProps {
  id: string;              // Required
  name: string;            // Required
  email?: string;          // Optional
  role?: 'admin' | 'user'; // Optional with union
}
```

### Children Pattern

```typescript
interface ContainerProps {
  children: React.ReactNode;
}

export function Container({ children }: ContainerProps) {
  return <div className="max-w-4xl mx-auto">{children}</div>;
}
```

### Render Prop

```typescript
interface DataLoaderProps<T> {
  fetcher: () => Promise<T>;
  render: (data: T) => React.ReactNode;
}

export function DataLoader<T>({ fetcher, render }: DataLoaderProps<T>) {
  const [data, setData] = useState<T | null>(null);

  useEffect(() => {
    fetcher().then(setData);
  }, []);

  if (!data) return <Loading />;
  return <>{render(data)}</>;
}
```

### Polymorphic Component

```typescript
type ButtonProps<T extends React.ElementType> = {
  as?: T;
  children: React.ReactNode;
} & React.ComponentPropsWithoutRef<T>;

export function Button<T extends React.ElementType = 'button'>({
  as,
  children,
  ...props
}: ButtonProps<T>) {
  const Component = as || 'button';
  return <Component {...props}>{children}</Component>;
}

// Usage
<Button>Click me</Button>
<Button as="a" href="/about">Link</Button>
```

## Composition Patterns

### Compound Components

```typescript
// components/Tabs/index.tsx
const TabsContext = createContext<{ active: string; setActive: (v: string) => void } | null>(null);

export function Tabs({ children, defaultValue }: { children: React.ReactNode; defaultValue: string }) {
  const [active, setActive] = useState(defaultValue);

  return (
    <TabsContext.Provider value={{ active, setActive }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

export function TabList({ children }: { children: React.ReactNode }) {
  return <div className="tab-list">{children}</div>;
}

export function Tab({ value, children }: { value: string; children: React.ReactNode }) {
  const context = useContext(TabsContext);
  if (!context) throw new Error('Tab must be used within Tabs');

  return (
    <button
      className={context.active === value ? 'active' : ''}
      onClick={() => context.setActive(value)}
    >
      {children}
    </button>
  );
}

export function TabPanel({ value, children }: { value: string; children: React.ReactNode }) {
  const context = useContext(TabsContext);
  if (!context) throw new Error('TabPanel must be used within Tabs');
  if (context.active !== value) return null;

  return <div className="tab-panel">{children}</div>;
}

// Usage
<Tabs defaultValue="tab1">
  <TabList>
    <Tab value="tab1">Tab 1</Tab>
    <Tab value="tab2">Tab 2</Tab>
  </TabList>
  <TabPanel value="tab1">Content 1</TabPanel>
  <TabPanel value="tab2">Content 2</TabPanel>
</Tabs>
```

## Loading & Error States

```typescript
interface AsyncContentProps<T> {
  isLoading: boolean;
  error: Error | null;
  data: T | undefined;
  children: (data: T) => React.ReactNode;
}

export function AsyncContent<T>({
  isLoading,
  error,
  data,
  children,
}: AsyncContentProps<T>) {
  if (isLoading) {
    return <div className="animate-pulse">Loading...</div>;
  }

  if (error) {
    return (
      <div className="text-red-500">
        Error: {error.message}
      </div>
    );
  }

  if (!data) {
    return <div>No data</div>;
  }

  return <>{children(data)}</>;
}
```

## Conditional Rendering

```typescript
// Short-circuit
{isLoggedIn && <UserMenu />}

// Ternary
{isLoggedIn ? <UserMenu /> : <LoginButton />}

// Early return
function UserProfile({ user }: { user: User | null }) {
  if (!user) return <div>Please log in</div>;

  return <div>{user.name}</div>;
}

// Multiple conditions
function Status({ status }: { status: 'loading' | 'error' | 'success' }) {
  const content = {
    loading: <Spinner />,
    error: <ErrorMessage />,
    success: <SuccessMessage />,
  };

  return content[status];
}
```

## Lists & Keys

```typescript
// Good: Stable unique ID
{users.map(user => (
  <UserCard key={user.id} user={user} />
))}

// Bad: Index as key (causes issues with reordering)
{users.map((user, index) => (
  <UserCard key={index} user={user} />
))}

// With fragment
{items.map(item => (
  <React.Fragment key={item.id}>
    <dt>{item.term}</dt>
    <dd>{item.description}</dd>
  </React.Fragment>
))}
```

## forwardRef

```typescript
import { forwardRef } from 'react';

interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
}

export const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ label, ...props }, ref) => {
    return (
      <div>
        <label>{label}</label>
        <input ref={ref} {...props} />
      </div>
    );
  }
);

Input.displayName = 'Input';
```

## cn() Utility (Class Merging)

```typescript
// lib/utils.ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

// Usage
<div className={cn(
  'base-class',
  isActive && 'active-class',
  className
)} />
```
