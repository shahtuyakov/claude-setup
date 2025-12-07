# Client-Side Authentication

## Token Storage Options

| Storage | Pros | Cons | Use When |
|---------|------|------|----------|
| httpOnly Cookie | XSS-safe, auto-sent | CSRF vulnerable | SSR apps, highest security |
| localStorage | Persistent, simple | XSS vulnerable | SPAs, less sensitive data |
| Memory (state) | Most secure | Lost on refresh | Short sessions, high security |

## Auth Context Pattern

### Setup

```typescript
// providers/AuthProvider.tsx
'use client';

import {
  createContext,
  useContext,
  useState,
  useEffect,
  ReactNode,
} from 'react';
import { useRouter } from 'next/navigation';

interface User {
  id: string;
  email: string;
  name: string;
  role: 'admin' | 'user';
}

interface AuthContextType {
  user: User | null;
  isLoading: boolean;
  isAuthenticated: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
  register: (data: RegisterData) => Promise<void>;
}

const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const router = useRouter();

  // Check auth on mount
  useEffect(() => {
    checkAuth();
  }, []);

  async function checkAuth() {
    try {
      const token = localStorage.getItem('token');
      if (!token) {
        setIsLoading(false);
        return;
      }

      const res = await fetch('/api/auth/me', {
        headers: { Authorization: `Bearer ${token}` },
      });

      if (res.ok) {
        const userData = await res.json();
        setUser(userData);
      } else {
        localStorage.removeItem('token');
      }
    } catch (error) {
      console.error('Auth check failed:', error);
    } finally {
      setIsLoading(false);
    }
  }

  async function login(email: string, password: string) {
    const res = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password }),
    });

    if (!res.ok) {
      const error = await res.json();
      throw new Error(error.message || 'Login failed');
    }

    const { user, token } = await res.json();
    localStorage.setItem('token', token);
    setUser(user);
    router.push('/dashboard');
  }

  async function logout() {
    localStorage.removeItem('token');
    setUser(null);
    router.push('/login');
  }

  async function register(data: RegisterData) {
    const res = await fetch('/api/auth/register', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });

    if (!res.ok) {
      const error = await res.json();
      throw new Error(error.message || 'Registration failed');
    }

    const { user, token } = await res.json();
    localStorage.setItem('token', token);
    setUser(user);
    router.push('/dashboard');
  }

  return (
    <AuthContext.Provider
      value={{
        user,
        isLoading,
        isAuthenticated: !!user,
        login,
        logout,
        register,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}
```

### Layout Integration

```typescript
// app/layout.tsx
import { AuthProvider } from '@/providers/AuthProvider';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <AuthProvider>{children}</AuthProvider>
      </body>
    </html>
  );
}
```

## Protected Routes

### Client-Side Guard

```typescript
// components/auth/ProtectedRoute.tsx
'use client';

import { useAuth } from '@/providers/AuthProvider';
import { useRouter } from 'next/navigation';
import { useEffect } from 'react';

interface ProtectedRouteProps {
  children: React.ReactNode;
  requiredRole?: 'admin' | 'user';
}

export function ProtectedRoute({ children, requiredRole }: ProtectedRouteProps) {
  const { user, isLoading, isAuthenticated } = useAuth();
  const router = useRouter();

  useEffect(() => {
    if (!isLoading && !isAuthenticated) {
      router.push('/login');
    }

    if (!isLoading && requiredRole && user?.role !== requiredRole) {
      router.push('/unauthorized');
    }
  }, [isLoading, isAuthenticated, user, requiredRole, router]);

  if (isLoading) {
    return <div>Loading...</div>;
  }

  if (!isAuthenticated) {
    return null;
  }

  if (requiredRole && user?.role !== requiredRole) {
    return null;
  }

  return <>{children}</>;
}

// Usage in page
export default function DashboardPage() {
  return (
    <ProtectedRoute>
      <Dashboard />
    </ProtectedRoute>
  );
}
```

### Protected Layout

```typescript
// app/(protected)/layout.tsx
'use client';

import { useAuth } from '@/providers/AuthProvider';
import { redirect } from 'next/navigation';

export default function ProtectedLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const { isAuthenticated, isLoading } = useAuth();

  if (isLoading) {
    return <div>Loading...</div>;
  }

  if (!isAuthenticated) {
    redirect('/login');
  }

  return <>{children}</>;
}
```

## Login Form

```typescript
'use client';

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { useAuth } from '@/providers/AuthProvider';
import { useState } from 'react';

const schema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(1, 'Password is required'),
});

type LoginData = z.infer<typeof schema>;

export function LoginForm() {
  const { login } = useAuth();
  const [error, setError] = useState<string | null>(null);

  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<LoginData>({
    resolver: zodResolver(schema),
  });

  const onSubmit = async (data: LoginData) => {
    try {
      setError(null);
      await login(data.email, data.password);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Login failed');
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      {error && (
        <div className="bg-red-50 text-red-500 p-3 rounded">{error}</div>
      )}

      <div>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          type="email"
          {...register('email')}
          className="w-full border rounded px-3 py-2"
        />
        {errors.email && (
          <p className="text-red-500 text-sm">{errors.email.message}</p>
        )}
      </div>

      <div>
        <label htmlFor="password">Password</label>
        <input
          id="password"
          type="password"
          {...register('password')}
          className="w-full border rounded px-3 py-2"
        />
        {errors.password && (
          <p className="text-red-500 text-sm">{errors.password.message}</p>
        )}
      </div>

      <button
        type="submit"
        disabled={isSubmitting}
        className="w-full bg-blue-500 text-white py-2 rounded disabled:opacity-50"
      >
        {isSubmitting ? 'Logging in...' : 'Login'}
      </button>
    </form>
  );
}
```

## API Client with Auth

```typescript
// lib/api-client.ts
class ApiClient {
  private baseUrl: string;

  constructor(baseUrl: string) {
    this.baseUrl = baseUrl;
  }

  private getToken(): string | null {
    if (typeof window === 'undefined') return null;
    return localStorage.getItem('token');
  }

  async fetch<T>(endpoint: string, options?: RequestInit): Promise<T> {
    const token = this.getToken();

    const res = await fetch(`${this.baseUrl}${endpoint}`, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...(token && { Authorization: `Bearer ${token}` }),
        ...options?.headers,
      },
    });

    if (res.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
      throw new Error('Unauthorized');
    }

    if (!res.ok) {
      const error = await res.json().catch(() => ({}));
      throw new Error(error.message || 'Request failed');
    }

    return res.json();
  }

  get<T>(endpoint: string) {
    return this.fetch<T>(endpoint);
  }

  post<T>(endpoint: string, data: unknown) {
    return this.fetch<T>(endpoint, {
      method: 'POST',
      body: JSON.stringify(data),
    });
  }

  put<T>(endpoint: string, data: unknown) {
    return this.fetch<T>(endpoint, {
      method: 'PUT',
      body: JSON.stringify(data),
    });
  }

  delete<T>(endpoint: string) {
    return this.fetch<T>(endpoint, { method: 'DELETE' });
  }
}

export const api = new ApiClient('/api');
```

## Token Refresh Pattern

```typescript
// lib/auth.ts
let refreshPromise: Promise<string> | null = null;

export async function getValidToken(): Promise<string | null> {
  const token = localStorage.getItem('token');
  const expiry = localStorage.getItem('tokenExpiry');

  if (!token) return null;

  // Check if token is about to expire (within 5 minutes)
  if (expiry && Date.now() > parseInt(expiry) - 5 * 60 * 1000) {
    // Prevent multiple refresh calls
    if (!refreshPromise) {
      refreshPromise = refreshToken();
    }
    return refreshPromise;
  }

  return token;
}

async function refreshToken(): Promise<string> {
  try {
    const refreshToken = localStorage.getItem('refreshToken');

    const res = await fetch('/api/auth/refresh', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ refreshToken }),
    });

    if (!res.ok) {
      throw new Error('Refresh failed');
    }

    const { token, expiresIn } = await res.json();

    localStorage.setItem('token', token);
    localStorage.setItem('tokenExpiry', String(Date.now() + expiresIn * 1000));

    return token;
  } finally {
    refreshPromise = null;
  }
}
```

## User Menu Component

```typescript
'use client';

import { useAuth } from '@/providers/AuthProvider';
import Link from 'next/link';

export function UserMenu() {
  const { user, logout, isAuthenticated } = useAuth();

  if (!isAuthenticated) {
    return (
      <div className="flex gap-2">
        <Link href="/login" className="text-blue-500">
          Login
        </Link>
        <Link href="/register" className="text-blue-500">
          Register
        </Link>
      </div>
    );
  }

  return (
    <div className="flex items-center gap-4">
      <span>{user?.name}</span>
      <button onClick={logout} className="text-red-500">
        Logout
      </button>
    </div>
  );
}
```

## Role-Based UI

```typescript
'use client';

import { useAuth } from '@/providers/AuthProvider';

interface RoleGateProps {
  children: React.ReactNode;
  allowedRoles: string[];
  fallback?: React.ReactNode;
}

export function RoleGate({ children, allowedRoles, fallback }: RoleGateProps) {
  const { user } = useAuth();

  if (!user || !allowedRoles.includes(user.role)) {
    return fallback || null;
  }

  return <>{children}</>;
}

// Usage
<RoleGate allowedRoles={['admin']}>
  <AdminPanel />
</RoleGate>

<RoleGate allowedRoles={['admin', 'moderator']} fallback={<p>Access denied</p>}>
  <ModeratorTools />
</RoleGate>
```

## Best Practices

1. **httpOnly cookies for tokens** - When possible, let backend set httpOnly cookies
2. **Handle token expiry** - Refresh before expiry, not after
3. **Clear state on logout** - Remove tokens and reset all auth state
4. **Show loading states** - Don't flash protected content
5. **Redirect after login** - Remember intended destination
6. **Handle 401 globally** - Redirect to login on unauthorized responses
