# Common Feature Breakdowns

Pre-analyzed task breakdowns for frequently requested features.

---

## User Authentication

**Complexity**: Medium

| Component | Tasks |
|-----------|-------|
| Database | users table (id, email, password_hash, created_at), refresh_tokens table |
| Backend | POST /register, POST /login, POST /logout, POST /refresh-token, password hashing (bcrypt) |
| Frontend | Registration form, login form, auth state management, token storage |

**Dependencies**: Database → Backend → Frontend

**iOS additions**: Keychain storage, biometric unlock option

---

## User Profile

**Complexity**: Simple-Medium

| Component | Tasks |
|-----------|-------|
| Database | Add profile fields to users (name, avatar_url, bio), or separate profiles table |
| Backend | GET /users/:id, PATCH /users/:id, avatar upload endpoint |
| Frontend | Profile view screen, edit profile form, avatar upload UI |

**Dependencies**: Auth must exist first, then Database → Backend → Frontend

---

## CRUD Resource (Generic)

**Complexity**: Simple

| Component | Tasks |
|-----------|-------|
| Database | Resource table with fields, indexes, foreign keys |
| Backend | GET /resources, GET /resources/:id, POST /resources, PATCH /resources/:id, DELETE /resources/:id |
| Frontend | List view, detail view, create form, edit form, delete confirmation |

**Dependencies**: Database → Backend → Frontend

---

## File/Image Upload

**Complexity**: Medium

| Component | Tasks |
|-----------|-------|
| Database | Store file metadata (url, filename, size, user_id) |
| Backend | Generate presigned URL endpoint, confirm upload endpoint, serve via CDN |
| Frontend | File picker, upload progress UI, display uploaded files |

**Dependencies**: Cloud storage setup first, then Database → Backend → Frontend

**iOS additions**: PHPickerViewController, upload with URLSession, image caching

---

## Payment Integration

**Complexity**: Complex

| Component | Tasks |
|-----------|-------|
| Database | payments table, subscriptions table (if recurring), webhook logs |
| Backend | Create checkout session, webhook handler, subscription management, receipt validation |
| Frontend | Payment button/form, subscription status UI, payment history |

**Dependencies**: Payment provider account first (Stripe), then Database → Backend → Frontend

**iOS additions**: StoreKit for in-app purchases (if App Store), or web-based Stripe checkout

---

## Real-time Chat

**Complexity**: Complex

| Component | Tasks |
|-----------|-------|
| Database | conversations table, messages table, participants table |
| Backend | WebSocket server, message endpoints, conversation CRUD, read receipts |
| Frontend | Conversation list, chat view, message input, real-time updates |

**Dependencies**: Auth first, then Database → Backend → Frontend (parallel WebSocket + REST)

**iOS additions**: WebSocket connection, APNs for background notifications, local message caching

---

## Push Notifications

**Complexity**: Medium

| Component | Tasks |
|-----------|-------|
| Database | device_tokens table (user_id, token, platform) |
| Backend | Register token endpoint, send notification service, APNs integration |
| Frontend | Request permission, register token with backend, handle notification tap |

**Dependencies**: Auth first, APNs setup, then Database → Backend → Frontend

**iOS additions**: UNUserNotificationCenter, token registration, notification handling

---

## Search

**Complexity**: Medium

| Component | Tasks |
|-----------|-------|
| Database | Full-text search indexes or Elasticsearch setup |
| Backend | GET /search?q=, search service, result ranking |
| Frontend | Search input, results display, filters, pagination |

**Dependencies**: Data must exist to search, then Database → Backend → Frontend

---

## Social Features (Follow/Like)

**Complexity**: Medium

| Component | Tasks |
|-----------|-------|
| Database | follows table (follower_id, following_id), likes table (user_id, resource_id) |
| Backend | POST/DELETE /follow/:userId, POST/DELETE /like/:resourceId, follower counts |
| Frontend | Follow/unfollow button, like button, follower/following lists |

**Dependencies**: Users must exist, then Database → Backend → Frontend

---

## Email Integration

**Complexity**: Simple-Medium

| Component | Tasks |
|-----------|-------|
| Database | email_logs table (optional, for tracking) |
| Backend | Email service integration (SendGrid, SES), email templates, send endpoints |
| Frontend | Usually none (backend-triggered), possibly email preferences UI |

**Dependencies**: Email provider setup, then Backend (no frontend usually)

---

## Admin Dashboard

**Complexity**: Medium-Complex

| Component | Tasks |
|-----------|-------|
| Database | Admin roles/permissions, audit logs |
| Backend | Admin-only endpoints, role middleware, analytics queries |
| Frontend | Separate admin UI, user management, content moderation, analytics views |

**Dependencies**: Auth + roles first, then Database → Backend → Frontend

---

## Third-party OAuth (Google, Apple)

**Complexity**: Medium

| Component | Tasks |
|-----------|-------|
| Database | oauth_accounts table (user_id, provider, provider_user_id) |
| Backend | OAuth callback endpoints, token exchange, account linking |
| Frontend | OAuth buttons, redirect handling, account linking UI |

**Dependencies**: Provider setup (credentials), then Database → Backend → Frontend

**iOS additions**: Sign in with Apple (required if offering other social logins), ASAuthorizationController

---

## Settings/Preferences

**Complexity**: Simple

| Component | Tasks |
|-----------|-------|
| Database | user_settings table or JSON column on users |
| Backend | GET/PATCH /settings |
| Frontend | Settings screen, toggle/input controls, save functionality |

**Dependencies**: Auth first, then Database → Backend → Frontend

**iOS additions**: UserDefaults for local cache, sync with backend
