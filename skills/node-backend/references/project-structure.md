# Project Structure

## Express / Fastify / Hono Structure

```
src/
├── index.ts              # Entry point
├── app.ts                # Express app setup
├── config/
│   ├── index.ts          # Config loader
│   ├── database.ts       # DB config
│   └── env.ts            # Environment validation
├── routes/
│   ├── index.ts          # Route aggregator
│   ├── auth.routes.ts    # Auth endpoints
│   ├── users.routes.ts   # User endpoints
│   └── health.routes.ts  # Health check
├── controllers/
│   ├── auth.controller.ts
│   └── users.controller.ts
├── services/
│   ├── auth.service.ts   # Business logic
│   ├── users.service.ts
│   └── email.service.ts
├── middleware/
│   ├── auth.middleware.ts
│   ├── validate.middleware.ts
│   └── error.middleware.ts
├── models/               # If using ORM
│   ├── user.model.ts
│   └── index.ts
├── repositories/         # Data access layer
│   └── user.repository.ts
├── schemas/              # Zod schemas
│   ├── auth.schema.ts
│   └── user.schema.ts
├── types/
│   ├── index.ts
│   └── express.d.ts      # Type extensions
└── utils/
    ├── logger.ts
    ├── jwt.ts
    └── hash.ts
```

## NestJS Structure

```
src/
├── main.ts               # Bootstrap
├── app.module.ts         # Root module
├── app.controller.ts     # Root controller
├── common/
│   ├── decorators/
│   ├── filters/
│   ├── guards/
│   ├── interceptors/
│   └── pipes/
├── config/
│   ├── config.module.ts
│   └── configuration.ts
├── auth/
│   ├── auth.module.ts
│   ├── auth.controller.ts
│   ├── auth.service.ts
│   ├── strategies/
│   │   ├── jwt.strategy.ts
│   │   └── local.strategy.ts
│   ├── guards/
│   │   └── jwt-auth.guard.ts
│   └── dto/
│       ├── login.dto.ts
│       └── register.dto.ts
├── users/
│   ├── users.module.ts
│   ├── users.controller.ts
│   ├── users.service.ts
│   ├── entities/
│   │   └── user.entity.ts
│   └── dto/
│       ├── create-user.dto.ts
│       └── update-user.dto.ts
└── database/
    ├── database.module.ts
    └── migrations/
```

## Layer Responsibilities

| Layer | Responsibility | Example |
|-------|---------------|---------|
| Routes/Controllers | HTTP handling, request/response | Parse params, call service, format response |
| Services | Business logic | Validate business rules, orchestrate operations |
| Repositories | Data access | Database queries, ORM operations |
| Models/Entities | Data structure | Schema definition, relationships |
| Middleware | Cross-cutting | Auth, logging, validation |
| Schemas | Validation | Input validation rules |

## File Naming Conventions

| Type | Pattern | Example |
|------|---------|---------|
| Routes | `[resource].routes.ts` | `users.routes.ts` |
| Controllers | `[resource].controller.ts` | `users.controller.ts` |
| Services | `[resource].service.ts` | `users.service.ts` |
| Models | `[resource].model.ts` | `user.model.ts` |
| Schemas | `[resource].schema.ts` | `user.schema.ts` |
| Middleware | `[name].middleware.ts` | `auth.middleware.ts` |
| Types | `[name].types.ts` | `auth.types.ts` |
| Tests | `[name].test.ts` or `[name].spec.ts` | `auth.service.test.ts` |

## Environment Files

```
.env                # Local development (gitignored)
.env.example        # Template (committed)
.env.test           # Test environment
.env.production     # Production (or use secrets manager)
```

## Config Pattern

```typescript
// src/config/env.ts
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  PORT: z.string().transform(Number).default('3000'),
  DATABASE_URL: z.string(),
  JWT_SECRET: z.string().min(32),
  JWT_EXPIRES_IN: z.string().default('15m'),
});

export const env = envSchema.parse(process.env);
```

## Entry Point Pattern

```typescript
// src/index.ts
import 'dotenv/config';
import { app } from './app';
import { env } from './config/env';
import { logger } from './utils/logger';
import { connectDatabase } from './config/database';

async function bootstrap() {
  await connectDatabase();

  app.listen(env.PORT, () => {
    logger.info(`Server running on port ${env.PORT}`);
  });
}

bootstrap().catch((error) => {
  logger.error('Failed to start server', error);
  process.exit(1);
});
```
