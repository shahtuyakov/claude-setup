# Authentication Patterns

## Quick Decision Matrix

| If you need... | Use | Avoid |
|----------------|-----|-------|
| Stateless API auth, mobile apps | JWT | Server sessions |
| Simple web app, server-rendered | Server sessions | JWT (overkill) |
| Third-party login (Google, Apple) | OAuth 2.0 / OIDC | Custom auth |
| iOS app authentication | JWT + Keychain + Biometrics | Storing tokens in UserDefaults |
| Machine-to-machine | API Keys or Client Credentials | User-based auth |

---

## JWT (JSON Web Tokens)

| Aspect | Details |
|--------|---------|
| **Best for** | Mobile apps, SPAs, stateless APIs, microservices |
| **Pros** | Stateless, scalable, can contain claims, works across services |
| **Cons** | Can't invalidate easily, token size, must handle refresh |
| **When to use** | iOS apps, REST/GraphQL APIs, distributed systems |
| **When to avoid** | When immediate logout/revocation is critical (use short expiry + refresh) |
| **Works well with** | OAuth 2.0, Redis (for refresh token storage/blacklist) |
| **iOS considerations** | Store in Keychain (never UserDefaults), implement refresh flow |

### JWT Best Practices
- Short access token expiry (15-60 minutes)
- Longer refresh token expiry (7-30 days)
- Store refresh tokens server-side (Redis/DB) for revocation
- Use RS256 for distributed systems, HS256 for single service
- Include minimal claims (user_id, roles) - not sensitive data

### JWT Token Flow (iOS)
```
1. Login → Backend returns access_token + refresh_token
2. Store both in Keychain
3. API calls → Authorization: Bearer {access_token}
4. On 401 → Use refresh_token to get new access_token
5. If refresh fails → Logout, re-authenticate
```

---

## Server Sessions

| Aspect | Details |
|--------|---------|
| **Best for** | Server-rendered web apps, simple applications |
| **Pros** | Easy to implement, immediate revocation, smaller payload |
| **Cons** | Stateful (sticky sessions or shared store), scaling complexity |
| **When to use** | Traditional web apps, when immediate logout needed |
| **When to avoid** | Mobile apps, microservices, high-scale stateless APIs |
| **Works well with** | Redis (session store), traditional frameworks (Rails, Django, Laravel) |
| **iOS considerations** | Not recommended for native apps - use JWT instead |

---

## OAuth 2.0 / OpenID Connect

| Aspect | Details |
|--------|---------|
| **Best for** | Third-party login, delegated authorization |
| **Pros** | Industry standard, secure delegation, supports SSO, Sign in with Apple |
| **Cons** | Complex to implement correctly, multiple flows to understand |
| **When to use** | Social login, Sign in with Apple (required for iOS), enterprise SSO |
| **When to avoid** | Simple apps with only email/password |
| **Works well with** | Auth0, Firebase Auth, custom implementation |
| **iOS considerations** | Sign in with Apple mandatory if offering other social logins |

### OAuth Flows

| Flow | Use Case |
|------|----------|
| Authorization Code + PKCE | Mobile apps, SPAs (recommended) |
| Client Credentials | Machine-to-machine |
| ~~Implicit~~ | Deprecated - don't use |
| ~~Password Grant~~ | Deprecated - don't use |

### Sign in with Apple (iOS)
- **Required** if app offers any third-party login
- Returns user ID, email (can be hidden), name
- Implement server-side token verification
- Handle "Hide My Email" relay addresses

---

## Biometric Authentication (iOS)

| Aspect | Details |
|--------|---------|
| **Best for** | Quick app unlock, transaction confirmation |
| **Pros** | Excellent UX, secure, native iOS support |
| **Cons** | Not a replacement for initial auth, device-specific |
| **When to use** | App unlock after initial login, confirming sensitive actions |
| **When to avoid** | As sole authentication method |
| **Works well with** | Keychain, JWT refresh flow |
| **iOS considerations** | LocalAuthentication framework, Face ID/Touch ID |

### Biometric Flow
```
1. Initial login → Email/password or OAuth
2. Store tokens in Keychain with biometric protection
3. App relaunch → Prompt biometric
4. Success → Retrieve tokens from Keychain
5. Failure → Fall back to full login
```

---

## iOS Keychain Storage

| Item | Storage | Protection |
|------|---------|------------|
| Access token | Keychain | `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` |
| Refresh token | Keychain | `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` |
| User credentials | Keychain + Biometric | `kSecAccessControlBiometryCurrentSet` |

### Keychain Best Practices
- Never store tokens in UserDefaults
- Use appropriate accessibility level
- Clear on logout
- Handle keychain errors gracefully
- Use keychain wrapper library for cleaner code

---

## Password Security (Backend)

| Aspect | Recommendation |
|--------|----------------|
| Hashing | bcrypt or Argon2 (never MD5/SHA1) |
| Salt | Automatic with bcrypt |
| Work factor | bcrypt cost 12+ |
| Min password length | 12 characters |
| Password rules | Length > complexity requirements |

---

## Token Refresh Strategy

| Strategy | Pros | Cons | Best For |
|----------|------|------|----------|
| Silent refresh | Seamless UX | Complexity | SPAs, mobile apps |
| Sliding expiry | Simple | Less secure | Low-risk apps |
| Refresh rotation | Most secure | Complexity | High-security apps |

### Refresh Token Rotation (Recommended for iOS)
```
1. Refresh request → Send current refresh_token
2. Backend validates → Issues new access_token + NEW refresh_token
3. Invalidate old refresh_token
4. If old refresh_token reused → Revoke all tokens (potential theft)
```

---

## Multi-Factor Authentication (MFA)

| Method | Security | UX | iOS Support |
|--------|----------|-----|-------------|
| SMS OTP | Low (SIM swap risk) | Good | Native |
| TOTP (Authenticator app) | High | Medium | Third-party apps |
| Push notification | High | Excellent | Custom implementation |
| Biometric | High | Excellent | Native |

---

## Auth Architecture Checklist

- [ ] Password hashing with bcrypt/Argon2
- [ ] JWT with short access token expiry
- [ ] Refresh token rotation implemented
- [ ] Tokens stored in Keychain (iOS)
- [ ] Biometric unlock option (iOS)
- [ ] Sign in with Apple (if social login offered)
- [ ] HTTPS only (no token over HTTP)
- [ ] Rate limiting on login endpoints
- [ ] Account lockout after failed attempts
- [ ] Secure password reset flow
