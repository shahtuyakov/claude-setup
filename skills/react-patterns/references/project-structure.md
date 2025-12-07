# Project Structure

## Next.js App Router (Recommended)

```
src/
├── app/                      # App Router
│   ├── layout.tsx            # Root layout
│   ├── page.tsx              # Home page
│   ├── loading.tsx           # Loading UI
│   ├── error.tsx             # Error UI
│   ├── not-found.tsx         # 404 page
│   ├── globals.css           # Global styles
│   ├── (auth)/               # Route group (no URL segment)
│   │   ├── login/
│   │   │   └── page.tsx
│   │   └── register/
│   │       └── page.tsx
│   ├── dashboard/
│   │   ├── layout.tsx        # Dashboard layout
│   │   ├── page.tsx          # /dashboard
│   │   └── settings/
│   │       └── page.tsx      # /dashboard/settings
│   └── api/                  # API routes
│       └── users/
│           └── route.ts
├── components/
│   ├── ui/                   # Reusable UI components
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   │   ├── Modal.tsx
│   │   └── index.ts
│   ├── forms/                # Form components
│   │   ├── LoginForm.tsx
│   │   └── RegisterForm.tsx
│   ├── layout/               # Layout components
│   │   ├── Header.tsx
│   │   ├── Footer.tsx
│   │   └── Sidebar.tsx
│   └── features/             # Feature-specific components
│       ├── users/
│       │   ├── UserCard.tsx
│       │   ├── UserList.tsx
│       │   └── index.ts
│       └── dashboard/
│           └── StatsCard.tsx
├── hooks/                    # Custom hooks
│   ├── useAuth.ts
│   ├── useLocalStorage.ts
│   └── index.ts
├── lib/                      # Utilities and configs
│   ├── api.ts                # API client
│   ├── utils.ts              # Helper functions
│   └── constants.ts
├── providers/                # Context providers
│   ├── AuthProvider.tsx
│   ├── QueryProvider.tsx
│   └── index.tsx
├── stores/                   # Zustand stores
│   └── useUserStore.ts
├── types/                    # TypeScript types
│   ├── user.ts
│   └── index.ts
└── styles/                   # Additional styles
    └── components.css
```

## Vite + React Structure

```
src/
├── main.tsx                  # Entry point
├── App.tsx                   # Root component
├── index.css                 # Global styles
├── vite-env.d.ts
├── components/
│   ├── ui/
│   └── features/
├── hooks/
├── lib/
├── pages/                    # Page components
│   ├── HomePage.tsx
│   ├── LoginPage.tsx
│   └── DashboardPage.tsx
├── providers/
├── router/                   # React Router config
│   └── index.tsx
├── stores/
└── types/
```

## File Naming Conventions

| Type | Pattern | Example |
|------|---------|---------|
| Components | PascalCase | `UserCard.tsx` |
| Hooks | camelCase with `use` | `useAuth.ts` |
| Utils | camelCase | `formatDate.ts` |
| Types | PascalCase | `User.ts` |
| Pages (App Router) | `page.tsx` | `app/users/page.tsx` |
| Layouts | `layout.tsx` | `app/dashboard/layout.tsx` |
| API Routes | `route.ts` | `app/api/users/route.ts` |

## Component File Structure

### Simple Component
```typescript
// components/ui/Button.tsx
interface ButtonProps {
  children: React.ReactNode;
  variant?: 'primary' | 'secondary';
  onClick?: () => void;
}

export function Button({ children, variant = 'primary', onClick }: ButtonProps) {
  return (
    <button className={`btn btn-${variant}`} onClick={onClick}>
      {children}
    </button>
  );
}
```

### Complex Component with Subcomponents
```
components/
└── Modal/
    ├── index.ts              # Export barrel
    ├── Modal.tsx             # Main component
    ├── ModalHeader.tsx       # Subcomponent
    ├── ModalBody.tsx
    ├── ModalFooter.tsx
    └── Modal.types.ts        # Types
```

## Index Exports (Barrel Files)

```typescript
// components/ui/index.ts
export { Button } from './Button';
export { Input } from './Input';
export { Modal } from './Modal';

// Usage
import { Button, Input, Modal } from '@/components/ui';
```

## Path Aliases

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@/components/*": ["./src/components/*"],
      "@/hooks/*": ["./src/hooks/*"],
      "@/lib/*": ["./src/lib/*"]
    }
  }
}
```

## App Router Special Files

| File | Purpose |
|------|---------|
| `page.tsx` | Route UI |
| `layout.tsx` | Shared layout |
| `loading.tsx` | Loading UI (Suspense) |
| `error.tsx` | Error boundary |
| `not-found.tsx` | 404 page |
| `route.ts` | API endpoint |
| `template.tsx` | Re-rendered layout |
| `default.tsx` | Parallel route fallback |

## Route Groups

```
app/
├── (marketing)/          # No URL impact
│   ├── page.tsx          # /
│   ├── about/
│   │   └── page.tsx      # /about
│   └── layout.tsx        # Marketing layout
├── (app)/                # No URL impact
│   ├── dashboard/
│   │   └── page.tsx      # /dashboard
│   └── layout.tsx        # App layout (with auth)
└── layout.tsx            # Root layout
```

## Environment Variables

```
.env.local                # Local dev (gitignored)
.env.development          # Dev defaults
.env.production           # Prod defaults
```

```typescript
// Access in code
// Server-side (any)
process.env.DATABASE_URL

// Client-side (must prefix with NEXT_PUBLIC_)
process.env.NEXT_PUBLIC_API_URL
```
