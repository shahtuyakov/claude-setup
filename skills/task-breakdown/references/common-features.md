# Common Feature Breakdowns

Pre-analyzed task breakdowns for frequently requested features.

## Agent Reference

| Agent | Skill | Focus |
|-------|-------|-------|
| Database | database-patterns | Schema, migrations, queries |
| Backend | node-backend | APIs, business logic |
| Frontend | react-patterns | React/Next.js web UI |
| iOS | swift-patterns | SwiftUI native apps |
| DevOps | devops-patterns | Docker, CI/CD, deployment |
| Designer | design-patterns | Tokens, themes, styling |

---

## User Authentication

**Complexity**: Medium
**Platform**: Web + iOS

| Agent | Tasks |
|-------|-------|
| Database | users table (id, email, password_hash, created_at), refresh_tokens table |
| Backend | POST /register, POST /login, POST /logout, POST /refresh-token, password hashing (argon2) |
| Frontend | Registration form, login form, auth state management, token storage (cookies) |
| iOS | Login/register screens, Keychain storage, biometric unlock (Face ID/Touch ID) |
| Designer | Auth form styling, error states, loading states |

**Sequence**: Database → Backend → Designer → Frontend + iOS (parallel)

---

## User Profile

**Complexity**: Simple-Medium
**Platform**: Web + iOS

| Agent | Tasks |
|-------|-------|
| Database | Add profile fields to users (name, avatar_url, bio), or separate profiles table |
| Backend | GET /users/:id, PATCH /users/:id, avatar upload endpoint |
| Frontend | Profile view screen, edit profile form, avatar upload UI |
| iOS | Profile screen, edit form, PHPickerViewController for avatar |
| Designer | Profile card design, avatar placeholder, edit form styling |

**Sequence**: Auth first → Database → Backend → Designer → Frontend + iOS

---

## CRUD Resource (Generic)

**Complexity**: Simple
**Platform**: Web + iOS

| Agent | Tasks |
|-------|-------|
| Database | Resource table with fields, indexes, foreign keys |
| Backend | GET /resources, GET /resources/:id, POST /resources, PATCH /resources/:id, DELETE /resources/:id |
| Frontend | List view, detail view, create form, edit form, delete confirmation |
| iOS | List view (NavigationStack), detail view, forms, swipe-to-delete |
| Designer | List item styling, form layout, empty states |

**Sequence**: Database → Backend → Designer → Frontend + iOS

---

## File/Image Upload

**Complexity**: Medium
**Platform**: Web + iOS

| Agent | Tasks |
|-------|-------|
| Database | Store file metadata (url, filename, size, user_id) |
| Backend | Generate presigned URL endpoint, confirm upload endpoint, serve via CDN |
| Frontend | File picker, upload progress UI, display uploaded files |
| iOS | PHPickerViewController, upload with URLSession, image caching (Kingfisher) |
| DevOps | S3/R2 bucket setup, CloudFront CDN configuration |

**Sequence**: DevOps (storage) → Database → Backend → Frontend + iOS

---

## Payment Integration

**Complexity**: Complex
**Platform**: Web + iOS

| Agent | Tasks |
|-------|-------|
| Database | payments table, subscriptions table (if recurring), webhook logs |
| Backend | Create checkout session, webhook handler, subscription management, receipt validation |
| Frontend | Payment button/form, subscription status UI, payment history |
| iOS | StoreKit 2 for in-app purchases, or web-based Stripe checkout via SFSafariViewController |
| Designer | Payment flow UI, subscription cards, pricing table |

**Sequence**: Provider setup (Stripe/App Store) → Database → Backend → Designer → Frontend + iOS

---

## Real-time Chat

**Complexity**: Complex
**Platform**: Web + iOS

| Agent | Tasks |
|-------|-------|
| Database | conversations table, messages table, participants table |
| Backend | WebSocket server, message endpoints, conversation CRUD, read receipts |
| Frontend | Conversation list, chat view, message input, real-time updates |
| iOS | WebSocket connection (URLSessionWebSocketTask), chat UI, local message caching (SwiftData) |
| Designer | Chat bubble design, typing indicators, message status icons |
| DevOps | WebSocket server deployment, Redis for pub/sub |

**Sequence**: Auth → Database → Backend + DevOps → Designer → Frontend + iOS

---

## Push Notifications

**Complexity**: Medium
**Platform**: Web + iOS

| Agent | Tasks |
|-------|-------|
| Database | device_tokens table (user_id, token, platform) |
| Backend | Register token endpoint, send notification service, APNs/FCM integration |
| Frontend | Request permission (web push), register token, handle notification click |
| iOS | UNUserNotificationCenter, token registration, notification handling, badge management |
| DevOps | APNs certificates, FCM setup, notification service deployment |

**Sequence**: DevOps (APNs/FCM) → Database → Backend → Frontend + iOS

---

## Search

**Complexity**: Medium
**Platform**: Web + iOS

| Agent | Tasks |
|-------|-------|
| Database | Full-text search indexes (PostgreSQL) or Elasticsearch/Meilisearch setup |
| Backend | GET /search?q=, search service, result ranking, filters |
| Frontend | Search input with debounce, results display, filters, pagination |
| iOS | UISearchController, search results list, recent searches (SwiftData) |
| Designer | Search bar styling, result cards, empty/no-results states |
| DevOps | Elasticsearch/Meilisearch deployment (if needed) |

**Sequence**: DevOps (search engine) → Database → Backend → Designer → Frontend + iOS

---

## Social Features (Follow/Like)

**Complexity**: Medium
**Platform**: Web + iOS

| Agent | Tasks |
|-------|-------|
| Database | follows table (follower_id, following_id), likes table (user_id, resource_id), counts |
| Backend | POST/DELETE /follow/:userId, POST/DELETE /like/:resourceId, follower counts |
| Frontend | Follow/unfollow button, like button with animation, follower/following lists |
| iOS | Follow button, like animation (scale spring), follower list screens |
| Designer | Follow button states, like animation, count badges |

**Sequence**: Users exist → Database → Backend → Designer → Frontend + iOS

---

## Email Integration

**Complexity**: Simple-Medium
**Platform**: Backend only

| Agent | Tasks |
|-------|-------|
| Database | email_logs table (optional, for tracking) |
| Backend | Email service integration (Resend, SendGrid), email templates (React Email), send endpoints |
| Frontend | Email preferences UI (optional) |
| DevOps | Email provider setup, DNS records (SPF, DKIM) |

**Sequence**: DevOps (provider) → Database → Backend → Frontend (optional)

---

## Admin Dashboard

**Complexity**: Medium-Complex
**Platform**: Web only

| Agent | Tasks |
|-------|-------|
| Database | Admin roles/permissions, audit logs table |
| Backend | Admin-only endpoints, role middleware, analytics queries |
| Frontend | Separate admin UI, user management, content moderation, analytics charts |
| Designer | Admin dashboard layout, data tables, charts styling |
| DevOps | Separate admin deployment (optional), access restrictions |

**Sequence**: Auth + roles → Database → Backend → Designer → Frontend

---

## Third-party OAuth (Google, Apple)

**Complexity**: Medium
**Platform**: Web + iOS

| Agent | Tasks |
|-------|-------|
| Database | oauth_accounts table (user_id, provider, provider_user_id) |
| Backend | OAuth callback endpoints, token exchange, account linking |
| Frontend | OAuth buttons (Google, Apple), redirect handling, account linking UI |
| iOS | Sign in with Apple (ASAuthorizationController), Google Sign-In SDK |
| Designer | OAuth button styling (follow brand guidelines) |

**Sequence**: Provider setup → Database → Backend → Designer → Frontend + iOS

**Note**: Sign in with Apple required on iOS if offering other social logins

---

## Settings/Preferences

**Complexity**: Simple
**Platform**: Web + iOS

| Agent | Tasks |
|-------|-------|
| Database | user_settings table or JSON column on users |
| Backend | GET/PATCH /settings |
| Frontend | Settings screen, toggle/input controls, save functionality |
| iOS | Settings screen (Form), UserDefaults for local cache, sync with backend |
| Designer | Settings list styling, toggle switches, section headers |

**Sequence**: Auth → Database → Backend → Designer → Frontend + iOS

---

## App Deployment (New Feature)

**Complexity**: Medium
**Platform**: DevOps

| Agent | Tasks |
|-------|-------|
| DevOps | Dockerfile, docker-compose, CI/CD pipeline (GitHub Actions), deployment config |
| Backend | Health check endpoint, graceful shutdown handling |
| Frontend | Build optimization, environment variables |

**Sequence**: Backend (health check) → DevOps

### Deployment Targets

| Target | DevOps Tasks |
|--------|--------------|
| Vercel | vercel.json, environment setup |
| Railway | railway.json, database plugin |
| Fly.io | fly.toml, scaling config |
| AWS ECS | Task definition, load balancer |
| Kubernetes | Manifests, Helm chart |

---

## Design System Setup (New Feature)

**Complexity**: Medium
**Platform**: Web + iOS

| Agent | Tasks |
|-------|-------|
| Designer | Design tokens (colors, typography, spacing), component patterns, dark mode |
| Frontend | Tailwind config, CSS variables, shadcn/ui setup |
| iOS | Color assets, typography styles, SwiftUI view modifiers |

**Sequence**: Designer → Frontend + iOS (parallel)

### Key Deliverables

- Color palette (OKLCH, light/dark modes)
- Typography scale (fluid)
- Spacing system (4px base)
- Component library setup
