# Error Handling

## Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": {
      "field": "email",
      "reason": "Invalid email format"
    }
  }
}
```

## Custom Error Classes

```typescript
// src/utils/errors.ts
export class AppError extends Error {
  constructor(
    public message: string,
    public statusCode: number,
    public code: string,
    public details?: Record<string, any>
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }

  toJSON() {
    return {
      error: {
        code: this.code,
        message: this.message,
        ...(this.details && { details: this.details }),
      },
    };
  }
}

export class BadRequestError extends AppError {
  constructor(message = 'Bad request', details?: Record<string, any>) {
    super(message, 400, 'BAD_REQUEST', details);
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(message, 401, 'UNAUTHORIZED');
  }
}

export class ForbiddenError extends AppError {
  constructor(message = 'Forbidden') {
    super(message, 403, 'FORBIDDEN');
  }
}

export class NotFoundError extends AppError {
  constructor(message = 'Resource not found') {
    super(message, 404, 'NOT_FOUND');
  }
}

export class ConflictError extends AppError {
  constructor(message = 'Resource already exists') {
    super(message, 409, 'CONFLICT');
  }
}

export class ValidationError extends AppError {
  constructor(message = 'Validation failed', details?: Record<string, any>) {
    super(message, 422, 'VALIDATION_ERROR', details);
  }
}

export class InternalError extends AppError {
  constructor(message = 'Internal server error') {
    super(message, 500, 'INTERNAL_ERROR');
  }
}
```

## Express Error Middleware

```typescript
// src/middleware/error.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { AppError } from '../utils/errors';
import { logger } from '../utils/logger';
import { ZodError } from 'zod';

export function errorHandler(
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
) {
  // Log error
  logger.error({
    message: err.message,
    stack: err.stack,
    path: req.path,
    method: req.method,
  });

  // Handle known errors
  if (err instanceof AppError) {
    return res.status(err.statusCode).json(err.toJSON());
  }

  // Handle Zod validation errors
  if (err instanceof ZodError) {
    return res.status(422).json({
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Invalid input data',
        details: err.errors.map((e) => ({
          field: e.path.join('.'),
          message: e.message,
        })),
      },
    });
  }

  // Handle JWT errors
  if (err.name === 'JsonWebTokenError') {
    return res.status(401).json({
      error: {
        code: 'INVALID_TOKEN',
        message: 'Invalid token',
      },
    });
  }

  if (err.name === 'TokenExpiredError') {
    return res.status(401).json({
      error: {
        code: 'TOKEN_EXPIRED',
        message: 'Token has expired',
      },
    });
  }

  // Handle unknown errors
  const isProduction = process.env.NODE_ENV === 'production';

  return res.status(500).json({
    error: {
      code: 'INTERNAL_ERROR',
      message: isProduction ? 'Internal server error' : err.message,
      ...(isProduction ? {} : { stack: err.stack }),
    },
  });
}

export function notFoundHandler(req: Request, res: Response) {
  res.status(404).json({
    error: {
      code: 'NOT_FOUND',
      message: `Route ${req.method} ${req.path} not found`,
    },
  });
}
```

## NestJS Exception Filter

```typescript
// src/common/filters/all-exceptions.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';
import { Response } from 'express';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name);

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest();

    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let code = 'INTERNAL_ERROR';
    let message = 'Internal server error';
    let details: any = undefined;

    if (exception instanceof HttpException) {
      status = exception.getStatus();
      const exceptionResponse = exception.getResponse() as any;

      code = this.getErrorCode(status);
      message = exceptionResponse.message || exception.message;

      if (Array.isArray(exceptionResponse.message)) {
        details = exceptionResponse.message;
      }
    }

    this.logger.error({
      message,
      status,
      path: request.url,
      method: request.method,
      stack: exception instanceof Error ? exception.stack : undefined,
    });

    response.status(status).json({
      error: {
        code,
        message: Array.isArray(message) ? message[0] : message,
        ...(details && { details }),
      },
    });
  }

  private getErrorCode(status: number): string {
    const codes: Record<number, string> = {
      400: 'BAD_REQUEST',
      401: 'UNAUTHORIZED',
      403: 'FORBIDDEN',
      404: 'NOT_FOUND',
      409: 'CONFLICT',
      422: 'VALIDATION_ERROR',
      429: 'TOO_MANY_REQUESTS',
      500: 'INTERNAL_ERROR',
    };
    return codes[status] || 'HTTP_ERROR';
  }
}
```

## Async Error Wrapper

```typescript
// src/utils/asyncHandler.ts
import { Request, Response, NextFunction, RequestHandler } from 'express';

type AsyncRequestHandler = (
  req: Request,
  res: Response,
  next: NextFunction
) => Promise<any>;

export function asyncHandler(fn: AsyncRequestHandler): RequestHandler {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
}

// Usage
router.get('/users', asyncHandler(async (req, res) => {
  const users = await usersService.findAll();
  res.json(users);
}));
```

## Error Codes Reference

| Code | Status | Description |
|------|--------|-------------|
| `BAD_REQUEST` | 400 | Invalid request format |
| `UNAUTHORIZED` | 401 | Authentication required |
| `INVALID_TOKEN` | 401 | Token is invalid |
| `TOKEN_EXPIRED` | 401 | Token has expired |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `CONFLICT` | 409 | Resource already exists |
| `VALIDATION_ERROR` | 422 | Input validation failed |
| `TOO_MANY_REQUESTS` | 429 | Rate limit exceeded |
| `INTERNAL_ERROR` | 500 | Server error |

## Logging Errors

```typescript
// src/utils/logger.ts
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport:
    process.env.NODE_ENV !== 'production'
      ? { target: 'pino-pretty' }
      : undefined,
});

// Usage in error handler
logger.error({
  err: error,
  req: {
    method: req.method,
    url: req.url,
    headers: req.headers,
  },
}, 'Request error');
```

## Database Error Handling

```typescript
// src/utils/db-errors.ts
import { AppError, ConflictError, InternalError } from './errors';

export function handleDatabaseError(error: any): never {
  // PostgreSQL unique violation
  if (error.code === '23505') {
    throw new ConflictError('Resource already exists');
  }

  // PostgreSQL foreign key violation
  if (error.code === '23503') {
    throw new AppError('Referenced resource not found', 400, 'FOREIGN_KEY_ERROR');
  }

  // Prisma unique constraint
  if (error.code === 'P2002') {
    const field = error.meta?.target?.[0] || 'field';
    throw new ConflictError(`${field} already exists`);
  }

  // Prisma not found
  if (error.code === 'P2025') {
    throw new AppError('Record not found', 404, 'NOT_FOUND');
  }

  throw new InternalError('Database error');
}
```
