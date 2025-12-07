---
name: node-backend
description: Node.js backend development patterns for Express, NestJS, Fastify, and Hono. Use when implementing APIs, authentication, middleware, services, and server-side logic with Node.js and TypeScript. Provides project structure, code patterns, and best practices.
---

# Node.js Backend

Patterns and best practices for Node.js backend development with TypeScript.

## Framework Selection

| Framework | Best For | Use When |
|-----------|----------|----------|
| Express | Simple APIs, flexibility | Quick setup, full control needed |
| NestJS | Enterprise, structured | Large teams, Angular-like structure |
| Fastify | Performance critical | High throughput, schema validation |
| Hono | Edge, lightweight | Cloudflare Workers, minimal overhead |

## Reference Files

| Topic | Load | Use When |
|-------|------|----------|
| Project structure | `references/project-structure.md` | Setting up new project or organizing code |
| Express patterns | `references/express-patterns.md` | Building with Express |
| NestJS patterns | `references/nestjs-patterns.md` | Building with NestJS |
| Auth implementation | `references/auth-patterns.md` | Implementing authentication |
| Error handling | `references/error-handling.md` | Setting up error handling |
| Validation | `references/validation.md` | Input validation patterns |

## Quick Start Patterns

### Express Basic Setup

```typescript
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import { errorHandler } from './middleware/errorHandler';
import { routes } from './routes';

const app = express();

app.use(helmet());
app.use(cors());
app.use(express.json());

app.use('/api', routes);
app.use(errorHandler);

export { app };
```

### NestJS Module Pattern

```typescript
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

## TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true
  }
}
```

## Essential Packages

| Package | Purpose |
|---------|---------|
| `typescript` | Type safety |
| `zod` | Runtime validation |
| `bcrypt` | Password hashing |
| `jsonwebtoken` | JWT handling |
| `helmet` | Security headers |
| `cors` | CORS handling |
| `winston` / `pino` | Logging |
| `prisma` / `drizzle` | Database ORM |

## Code Quality Rules

1. **Always use TypeScript** - strict mode enabled
2. **Validate all inputs** - use Zod or class-validator
3. **Handle all errors** - never swallow exceptions
4. **Use async/await** - no callback hell
5. **Environment variables** - use dotenv, never hardcode secrets
6. **Logging** - structured logs with correlation IDs
