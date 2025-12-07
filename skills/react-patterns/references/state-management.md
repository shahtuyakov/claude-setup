# State Management

## When to Use What

| State Type | Solution | Use When |
|------------|----------|----------|
| Server state | React Query / SWR | Data from API, caching needed |
| Client global | Zustand | Shared UI state, user preferences |
| Component local | useState | Single component state |
| Form state | React Hook Form | Form inputs, validation |
| URL state | useSearchParams | Filters, pagination, shareable state |

## Zustand (Recommended for Client State)

### Setup

```bash
npm install zustand
```

### Basic Store

```typescript
// stores/useUserStore.ts
import { create } from 'zustand';

interface User {
  id: string;
  name: string;
  email: string;
}

interface UserState {
  user: User | null;
  isAuthenticated: boolean;
  setUser: (user: User) => void;
  logout: () => void;
}

export const useUserStore = create<UserState>((set) => ({
  user: null,
  isAuthenticated: false,
  setUser: (user) => set({ user, isAuthenticated: true }),
  logout: () => set({ user: null, isAuthenticated: false }),
}));
```

### Usage in Components

```typescript
'use client';

import { useUserStore } from '@/stores/useUserStore';

export function UserProfile() {
  const { user, logout } = useUserStore();

  if (!user) return null;

  return (
    <div>
      <p>{user.name}</p>
      <button onClick={logout}>Logout</button>
    </div>
  );
}
```

### Store with Persistence

```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface ThemeState {
  theme: 'light' | 'dark';
  toggleTheme: () => void;
}

export const useThemeStore = create<ThemeState>()(
  persist(
    (set) => ({
      theme: 'light',
      toggleTheme: () =>
        set((state) => ({
          theme: state.theme === 'light' ? 'dark' : 'light',
        })),
    }),
    { name: 'theme-storage' }
  )
);
```

### Store with Computed Values

```typescript
import { create } from 'zustand';

interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

interface CartState {
  items: CartItem[];
  addItem: (item: Omit<CartItem, 'quantity'>) => void;
  removeItem: (id: string) => void;
  clearCart: () => void;
  // Computed
  totalItems: () => number;
  totalPrice: () => number;
}

export const useCartStore = create<CartState>((set, get) => ({
  items: [],
  addItem: (item) =>
    set((state) => {
      const existing = state.items.find((i) => i.id === item.id);
      if (existing) {
        return {
          items: state.items.map((i) =>
            i.id === item.id ? { ...i, quantity: i.quantity + 1 } : i
          ),
        };
      }
      return { items: [...state.items, { ...item, quantity: 1 }] };
    }),
  removeItem: (id) =>
    set((state) => ({
      items: state.items.filter((i) => i.id !== id),
    })),
  clearCart: () => set({ items: [] }),
  totalItems: () => get().items.reduce((acc, item) => acc + item.quantity, 0),
  totalPrice: () =>
    get().items.reduce((acc, item) => acc + item.price * item.quantity, 0),
}));
```

### Async Actions

```typescript
interface UserState {
  user: User | null;
  loading: boolean;
  error: string | null;
  fetchUser: (id: string) => Promise<void>;
}

export const useUserStore = create<UserState>((set) => ({
  user: null,
  loading: false,
  error: null,
  fetchUser: async (id) => {
    set({ loading: true, error: null });
    try {
      const res = await fetch(`/api/users/${id}`);
      const user = await res.json();
      set({ user, loading: false });
    } catch (error) {
      set({ error: 'Failed to fetch user', loading: false });
    }
  },
}));
```

## React Context (Simple Global State)

### When to Use Context

- Theme/locale preferences
- Auth state (simple apps)
- Feature flags
- Avoid for: Frequently updating data

### Context Pattern

```typescript
// providers/ThemeProvider.tsx
'use client';

import { createContext, useContext, useState, ReactNode } from 'react';

type Theme = 'light' | 'dark';

interface ThemeContextType {
  theme: Theme;
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextType | null>(null);

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<Theme>('light');

  const toggleTheme = () => {
    setTheme((prev) => (prev === 'light' ? 'dark' : 'light'));
  };

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return context;
}
```

### Setup in Layout

```typescript
// app/layout.tsx
import { ThemeProvider } from '@/providers/ThemeProvider';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <ThemeProvider>{children}</ThemeProvider>
      </body>
    </html>
  );
}
```

## URL State (useSearchParams)

### Filter Pattern

```typescript
'use client';

import { useSearchParams, useRouter, usePathname } from 'next/navigation';

export function ProductFilters() {
  const searchParams = useSearchParams();
  const router = useRouter();
  const pathname = usePathname();

  const updateFilter = (key: string, value: string) => {
    const params = new URLSearchParams(searchParams.toString());
    if (value) {
      params.set(key, value);
    } else {
      params.delete(key);
    }
    router.push(`${pathname}?${params.toString()}`);
  };

  return (
    <div>
      <select
        value={searchParams.get('category') || ''}
        onChange={(e) => updateFilter('category', e.target.value)}
      >
        <option value="">All Categories</option>
        <option value="electronics">Electronics</option>
        <option value="clothing">Clothing</option>
      </select>
    </div>
  );
}
```

## Local Component State

### useState Patterns

```typescript
// Simple state
const [count, setCount] = useState(0);

// Object state (replace entire object)
const [user, setUser] = useState({ name: '', email: '' });
setUser({ ...user, name: 'John' });

// Functional update (for derived state)
setCount((prev) => prev + 1);

// Lazy initialization (expensive computation)
const [data, setData] = useState(() => computeExpensiveValue());
```

### useReducer (Complex Local State)

```typescript
type State = { count: number; step: number };
type Action =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'setStep'; payload: number };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment':
      return { ...state, count: state.count + state.step };
    case 'decrement':
      return { ...state, count: state.count - state.step };
    case 'setStep':
      return { ...state, step: action.payload };
    default:
      return state;
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0, step: 1 });

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
    </div>
  );
}
```

## Best Practices

1. **Colocate state** - Keep state close to where it's used
2. **Lift state up** - Only when siblings need to share
3. **Avoid prop drilling** - Use context or Zustand for deep trees
4. **Server state ≠ client state** - Use React Query for API data
5. **URL for shareable state** - Filters, pagination, search
