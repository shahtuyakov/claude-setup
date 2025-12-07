# Auth Patterns

## JWT Authentication Flow

```
1. Register: POST /auth/register → Create user, return tokens
2. Login: POST /auth/login → Verify credentials, return tokens
3. Access: Authorization: Bearer {access_token}
4. Refresh: POST /auth/refresh → Exchange refresh_token for new tokens
5. Logout: POST /auth/logout → Invalidate refresh_token
```

## Token Strategy

| Token | Expiry | Storage (Client) | Storage (Server) |
|-------|--------|------------------|------------------|
| Access token | 15-60 min | Memory / Keychain | Not stored |
| Refresh token | 7-30 days | Keychain only | Database/Redis |

## Password Hashing

```typescript
// src/utils/hash.ts
import bcrypt from 'bcrypt';

const SALT_ROUNDS = 12;

export async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS);
}

export async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return bcrypt.compare(password, hash);
}
```

## JWT Utilities

```typescript
// src/utils/jwt.ts
import jwt from 'jsonwebtoken';
import { env } from '../config/env';

export interface TokenPayload {
  userId: string;
  email: string;
}

export interface TokenPair {
  accessToken: string;
  refreshToken: string;
}

export function generateTokens(payload: TokenPayload): TokenPair {
  const accessToken = jwt.sign(payload, env.JWT_SECRET, {
    expiresIn: env.JWT_ACCESS_EXPIRES, // '15m'
  });

  const refreshToken = jwt.sign(payload, env.JWT_REFRESH_SECRET, {
    expiresIn: env.JWT_REFRESH_EXPIRES, // '7d'
  });

  return { accessToken, refreshToken };
}

export function verifyAccessToken(token: string): TokenPayload {
  return jwt.verify(token, env.JWT_SECRET) as TokenPayload;
}

export function verifyRefreshToken(token: string): TokenPayload {
  return jwt.verify(token, env.JWT_REFRESH_SECRET) as TokenPayload;
}
```

## Auth Service (Express)

```typescript
// src/services/auth.service.ts
import { usersRepository } from '../repositories/users.repository';
import { refreshTokensRepository } from '../repositories/refresh-tokens.repository';
import { hashPassword, verifyPassword } from '../utils/hash';
import { generateTokens, verifyRefreshToken } from '../utils/jwt';
import { UnauthorizedError, ConflictError } from '../utils/errors';
import { RegisterDto, LoginDto } from '../schemas/auth.schema';

class AuthService {
  async register(data: RegisterDto) {
    const existing = await usersRepository.findByEmail(data.email);
    if (existing) {
      throw new ConflictError('Email already registered');
    }

    const hashedPassword = await hashPassword(data.password);
    const user = await usersRepository.create({
      email: data.email,
      password: hashedPassword,
      name: data.name,
    });

    const tokens = generateTokens({ userId: user.id, email: user.email });

    await refreshTokensRepository.create({
      userId: user.id,
      token: tokens.refreshToken,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7 days
    });

    return {
      user: { id: user.id, email: user.email, name: user.name },
      ...tokens,
    };
  }

  async login(data: LoginDto) {
    const user = await usersRepository.findByEmail(data.email);
    if (!user) {
      throw new UnauthorizedError('Invalid credentials');
    }

    const isValid = await verifyPassword(data.password, user.password);
    if (!isValid) {
      throw new UnauthorizedError('Invalid credentials');
    }

    const tokens = generateTokens({ userId: user.id, email: user.email });

    await refreshTokensRepository.create({
      userId: user.id,
      token: tokens.refreshToken,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    });

    return {
      user: { id: user.id, email: user.email, name: user.name },
      ...tokens,
    };
  }

  async refresh(refreshToken: string) {
    try {
      const payload = verifyRefreshToken(refreshToken);

      // Check if token exists in database (not revoked)
      const storedToken = await refreshTokensRepository.findByToken(refreshToken);
      if (!storedToken) {
        throw new UnauthorizedError('Invalid refresh token');
      }

      // Delete old refresh token
      await refreshTokensRepository.delete(storedToken.id);

      // Generate new token pair
      const tokens = generateTokens({ userId: payload.userId, email: payload.email });

      // Store new refresh token
      await refreshTokensRepository.create({
        userId: payload.userId,
        token: tokens.refreshToken,
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
      });

      return tokens;
    } catch (error) {
      throw new UnauthorizedError('Invalid refresh token');
    }
  }

  async logout(refreshToken: string) {
    await refreshTokensRepository.deleteByToken(refreshToken);
  }

  async logoutAll(userId: string) {
    await refreshTokensRepository.deleteAllByUserId(userId);
  }
}

export const authService = new AuthService();
```

## Auth Controller (Express)

```typescript
// src/controllers/auth.controller.ts
import { Request, Response, NextFunction } from 'express';
import { authService } from '../services/auth.service';

class AuthController {
  async register(req: Request, res: Response, next: NextFunction) {
    try {
      const result = await authService.register(req.body);
      res.status(201).json(result);
    } catch (error) {
      next(error);
    }
  }

  async login(req: Request, res: Response, next: NextFunction) {
    try {
      const result = await authService.login(req.body);
      res.json(result);
    } catch (error) {
      next(error);
    }
  }

  async refresh(req: Request, res: Response, next: NextFunction) {
    try {
      const { refreshToken } = req.body;
      const tokens = await authService.refresh(refreshToken);
      res.json(tokens);
    } catch (error) {
      next(error);
    }
  }

  async logout(req: Request, res: Response, next: NextFunction) {
    try {
      const { refreshToken } = req.body;
      await authService.logout(refreshToken);
      res.status(204).send();
    } catch (error) {
      next(error);
    }
  }

  async me(req: Request, res: Response, next: NextFunction) {
    try {
      res.json(req.user);
    } catch (error) {
      next(error);
    }
  }
}

export const authController = new AuthController();
```

## Auth Middleware

```typescript
// src/middleware/auth.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { verifyAccessToken } from '../utils/jwt';
import { usersRepository } from '../repositories/users.repository';
import { UnauthorizedError } from '../utils/errors';

export async function authenticate(req: Request, res: Response, next: NextFunction) {
  try {
    const authHeader = req.headers.authorization;

    if (!authHeader?.startsWith('Bearer ')) {
      throw new UnauthorizedError('No token provided');
    }

    const token = authHeader.slice(7);
    const payload = verifyAccessToken(token);

    const user = await usersRepository.findById(payload.userId);
    if (!user) {
      throw new UnauthorizedError('User not found');
    }

    req.user = user;
    next();
  } catch (error) {
    next(new UnauthorizedError('Invalid token'));
  }
}

// Optional auth - doesn't fail if no token
export async function optionalAuth(req: Request, res: Response, next: NextFunction) {
  try {
    const authHeader = req.headers.authorization;

    if (authHeader?.startsWith('Bearer ')) {
      const token = authHeader.slice(7);
      const payload = verifyAccessToken(token);
      const user = await usersRepository.findById(payload.userId);
      req.user = user;
    }

    next();
  } catch {
    next(); // Continue without user
  }
}
```

## Auth Routes

```typescript
// src/routes/auth.routes.ts
import { Router } from 'express';
import { authController } from '../controllers/auth.controller';
import { authenticate } from '../middleware/auth.middleware';
import { validate } from '../middleware/validate.middleware';
import { registerSchema, loginSchema, refreshSchema } from '../schemas/auth.schema';

const router = Router();

router.post('/register', validate(registerSchema), authController.register);
router.post('/login', validate(loginSchema), authController.login);
router.post('/refresh', validate(refreshSchema), authController.refresh);
router.post('/logout', authController.logout);
router.get('/me', authenticate, authController.me);

export { router as authRoutes };
```

## Auth Schemas

```typescript
// src/schemas/auth.schema.ts
import { z } from 'zod';

export const registerSchema = z.object({
  body: z.object({
    email: z.string().email('Invalid email'),
    password: z.string().min(8, 'Password must be at least 8 characters'),
    name: z.string().optional(),
  }),
});

export const loginSchema = z.object({
  body: z.object({
    email: z.string().email('Invalid email'),
    password: z.string().min(1, 'Password is required'),
  }),
});

export const refreshSchema = z.object({
  body: z.object({
    refreshToken: z.string().min(1, 'Refresh token is required'),
  }),
});

export type RegisterDto = z.infer<typeof registerSchema>['body'];
export type LoginDto = z.infer<typeof loginSchema>['body'];
```

## OAuth (Google) Pattern

```typescript
// src/services/oauth.service.ts
import { OAuth2Client } from 'google-auth-library';
import { usersRepository } from '../repositories/users.repository';
import { oauthAccountsRepository } from '../repositories/oauth-accounts.repository';
import { generateTokens } from '../utils/jwt';

const googleClient = new OAuth2Client(process.env.GOOGLE_CLIENT_ID);

class OAuthService {
  async googleLogin(idToken: string) {
    const ticket = await googleClient.verifyIdToken({
      idToken,
      audience: process.env.GOOGLE_CLIENT_ID,
    });

    const payload = ticket.getPayload();
    if (!payload?.email) {
      throw new Error('Invalid Google token');
    }

    // Find or create user
    let user = await usersRepository.findByEmail(payload.email);

    if (!user) {
      user = await usersRepository.create({
        email: payload.email,
        name: payload.name,
        isVerified: payload.email_verified,
      });
    }

    // Link OAuth account
    await oauthAccountsRepository.upsert({
      userId: user.id,
      provider: 'google',
      providerUserId: payload.sub,
    });

    const tokens = generateTokens({ userId: user.id, email: user.email });

    return {
      user: { id: user.id, email: user.email, name: user.name },
      ...tokens,
    };
  }
}

export const oauthService = new OAuthService();
```

## Security Checklist

- [ ] Passwords hashed with bcrypt (cost 12+)
- [ ] JWT secrets are strong (32+ chars)
- [ ] Access tokens short-lived (15-60 min)
- [ ] Refresh tokens stored server-side
- [ ] Refresh token rotation implemented
- [ ] HTTPS only in production
- [ ] Rate limiting on auth endpoints
- [ ] Account lockout after failed attempts
- [ ] Secure password reset flow
- [ ] Input validation on all endpoints
