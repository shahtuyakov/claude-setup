# Express Patterns

## App Setup

```typescript
// src/app.ts
import express, { Express } from 'express';
import cors from 'cors';
import helmet from 'helmet';
import compression from 'compression';
import { routes } from './routes';
import { errorHandler, notFoundHandler } from './middleware/error.middleware';
import { requestLogger } from './middleware/logger.middleware';

export function createApp(): Express {
  const app = express();

  // Security
  app.use(helmet());
  app.use(cors({
    origin: process.env.CORS_ORIGIN || '*',
    credentials: true,
  }));

  // Parsing
  app.use(express.json({ limit: '10kb' }));
  app.use(express.urlencoded({ extended: true }));
  app.use(compression());

  // Logging
  app.use(requestLogger);

  // Health check
  app.get('/health', (req, res) => res.json({ status: 'ok' }));

  // Routes
  app.use('/api/v1', routes);

  // Error handling (must be last)
  app.use(notFoundHandler);
  app.use(errorHandler);

  return app;
}

export const app = createApp();
```

## Route Organization

```typescript
// src/routes/index.ts
import { Router } from 'express';
import { authRoutes } from './auth.routes';
import { usersRoutes } from './users.routes';
import { postsRoutes } from './posts.routes';

const router = Router();

router.use('/auth', authRoutes);
router.use('/users', usersRoutes);
router.use('/posts', postsRoutes);

export { router as routes };
```

```typescript
// src/routes/users.routes.ts
import { Router } from 'express';
import { usersController } from '../controllers/users.controller';
import { authenticate } from '../middleware/auth.middleware';
import { validate } from '../middleware/validate.middleware';
import { createUserSchema, updateUserSchema } from '../schemas/user.schema';

const router = Router();

router.get('/', authenticate, usersController.getAll);
router.get('/:id', authenticate, usersController.getById);
router.post('/', validate(createUserSchema), usersController.create);
router.patch('/:id', authenticate, validate(updateUserSchema), usersController.update);
router.delete('/:id', authenticate, usersController.delete);

export { router as usersRoutes };
```

## Controller Pattern

```typescript
// src/controllers/users.controller.ts
import { Request, Response, NextFunction } from 'express';
import { usersService } from '../services/users.service';
import { CreateUserDto, UpdateUserDto } from '../schemas/user.schema';

class UsersController {
  async getAll(req: Request, res: Response, next: NextFunction) {
    try {
      const { page = 1, limit = 10 } = req.query;
      const users = await usersService.findAll({
        page: Number(page),
        limit: Number(limit),
      });
      res.json(users);
    } catch (error) {
      next(error);
    }
  }

  async getById(req: Request, res: Response, next: NextFunction) {
    try {
      const user = await usersService.findById(req.params.id);
      res.json(user);
    } catch (error) {
      next(error);
    }
  }

  async create(req: Request, res: Response, next: NextFunction) {
    try {
      const data: CreateUserDto = req.body;
      const user = await usersService.create(data);
      res.status(201).json(user);
    } catch (error) {
      next(error);
    }
  }

  async update(req: Request, res: Response, next: NextFunction) {
    try {
      const data: UpdateUserDto = req.body;
      const user = await usersService.update(req.params.id, data);
      res.json(user);
    } catch (error) {
      next(error);
    }
  }

  async delete(req: Request, res: Response, next: NextFunction) {
    try {
      await usersService.delete(req.params.id);
      res.status(204).send();
    } catch (error) {
      next(error);
    }
  }
}

export const usersController = new UsersController();
```

## Service Pattern

```typescript
// src/services/users.service.ts
import { usersRepository } from '../repositories/users.repository';
import { CreateUserDto, UpdateUserDto } from '../schemas/user.schema';
import { NotFoundError, ConflictError } from '../utils/errors';
import { hashPassword } from '../utils/hash';

class UsersService {
  async findAll(options: { page: number; limit: number }) {
    const { page, limit } = options;
    const offset = (page - 1) * limit;

    const [users, total] = await Promise.all([
      usersRepository.findMany({ offset, limit }),
      usersRepository.count(),
    ]);

    return {
      data: users,
      meta: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit),
      },
    };
  }

  async findById(id: string) {
    const user = await usersRepository.findById(id);
    if (!user) {
      throw new NotFoundError('User not found');
    }
    return user;
  }

  async create(data: CreateUserDto) {
    const existing = await usersRepository.findByEmail(data.email);
    if (existing) {
      throw new ConflictError('Email already registered');
    }

    const hashedPassword = await hashPassword(data.password);
    return usersRepository.create({
      ...data,
      password: hashedPassword,
    });
  }

  async update(id: string, data: UpdateUserDto) {
    const user = await this.findById(id);

    if (data.password) {
      data.password = await hashPassword(data.password);
    }

    return usersRepository.update(id, data);
  }

  async delete(id: string) {
    await this.findById(id);
    return usersRepository.delete(id);
  }
}

export const usersService = new UsersService();
```

## Async Handler Wrapper

```typescript
// src/utils/asyncHandler.ts
import { Request, Response, NextFunction, RequestHandler } from 'express';

export function asyncHandler(
  fn: (req: Request, res: Response, next: NextFunction) => Promise<any>
): RequestHandler {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
}

// Usage in routes
router.get('/', asyncHandler(async (req, res) => {
  const users = await usersService.findAll();
  res.json(users);
}));
```

## Request Type Extension

```typescript
// src/types/express.d.ts
import { User } from '../models/user.model';

declare global {
  namespace Express {
    interface Request {
      user?: User;
    }
  }
}
```

## Pagination Helper

```typescript
// src/utils/pagination.ts
export interface PaginationParams {
  page?: number;
  limit?: number;
}

export interface PaginatedResult<T> {
  data: T[];
  meta: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
  };
}

export function getPaginationParams(query: any): { offset: number; limit: number } {
  const page = Math.max(1, parseInt(query.page) || 1);
  const limit = Math.min(100, Math.max(1, parseInt(query.limit) || 10));
  const offset = (page - 1) * limit;

  return { offset, limit };
}
```

## Graceful Shutdown

```typescript
// src/index.ts
import { app } from './app';
import { logger } from './utils/logger';
import { closeDatabase } from './config/database';

const server = app.listen(process.env.PORT, () => {
  logger.info(`Server running on port ${process.env.PORT}`);
});

async function shutdown(signal: string) {
  logger.info(`${signal} received, shutting down gracefully`);

  server.close(async () => {
    logger.info('HTTP server closed');
    await closeDatabase();
    logger.info('Database connection closed');
    process.exit(0);
  });

  // Force exit after 10s
  setTimeout(() => {
    logger.error('Forced shutdown after timeout');
    process.exit(1);
  }, 10000);
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```
