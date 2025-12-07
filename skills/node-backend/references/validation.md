# Validation Patterns

## Zod (Recommended for Express/Fastify)

### Setup

```bash
npm install zod
```

### Schema Definition

```typescript
// src/schemas/user.schema.ts
import { z } from 'zod';

export const createUserSchema = z.object({
  body: z.object({
    email: z.string().email('Invalid email format'),
    password: z
      .string()
      .min(8, 'Password must be at least 8 characters')
      .regex(/[A-Z]/, 'Password must contain uppercase letter')
      .regex(/[0-9]/, 'Password must contain number'),
    name: z.string().min(2).max(100).optional(),
    age: z.number().int().min(18).max(120).optional(),
  }),
});

export const updateUserSchema = z.object({
  body: z.object({
    email: z.string().email().optional(),
    name: z.string().min(2).max(100).optional(),
    age: z.number().int().min(18).max(120).optional(),
  }),
  params: z.object({
    id: z.string().uuid('Invalid user ID'),
  }),
});

export const getUserSchema = z.object({
  params: z.object({
    id: z.string().uuid('Invalid user ID'),
  }),
});

export const listUsersSchema = z.object({
  query: z.object({
    page: z.string().regex(/^\d+$/).transform(Number).default('1'),
    limit: z.string().regex(/^\d+$/).transform(Number).default('10'),
    search: z.string().optional(),
    sortBy: z.enum(['createdAt', 'name', 'email']).default('createdAt'),
    order: z.enum(['asc', 'desc']).default('desc'),
  }),
});

// Export types
export type CreateUserDto = z.infer<typeof createUserSchema>['body'];
export type UpdateUserDto = z.infer<typeof updateUserSchema>['body'];
export type ListUsersQuery = z.infer<typeof listUsersSchema>['query'];
```

### Validation Middleware (Express)

```typescript
// src/middleware/validate.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { AnyZodObject, ZodError } from 'zod';

export function validate(schema: AnyZodObject) {
  return async (req: Request, res: Response, next: NextFunction) => {
    try {
      await schema.parseAsync({
        body: req.body,
        query: req.query,
        params: req.params,
      });
      next();
    } catch (error) {
      if (error instanceof ZodError) {
        return res.status(422).json({
          error: {
            code: 'VALIDATION_ERROR',
            message: 'Invalid input data',
            details: error.errors.map((e) => ({
              field: e.path.join('.'),
              message: e.message,
            })),
          },
        });
      }
      next(error);
    }
  };
}

// Usage in routes
router.post('/users', validate(createUserSchema), usersController.create);
router.patch('/users/:id', validate(updateUserSchema), usersController.update);
```

### Common Zod Patterns

```typescript
// Email
z.string().email()

// URL
z.string().url()

// UUID
z.string().uuid()

// Enum
z.enum(['active', 'inactive', 'pending'])

// Date
z.string().datetime() // ISO string
z.coerce.date() // Coerce to Date object

// Array
z.array(z.string()).min(1).max(10)

// Object with unknown keys
z.record(z.string(), z.any())

// Union
z.union([z.string(), z.number()])

// Discriminated union
z.discriminatedUnion('type', [
  z.object({ type: z.literal('email'), email: z.string().email() }),
  z.object({ type: z.literal('phone'), phone: z.string() }),
])

// Transform
z.string().transform((val) => val.toLowerCase())

// Refine (custom validation)
z.string().refine((val) => !val.includes(' '), {
  message: 'Cannot contain spaces',
})

// Optional with default
z.string().optional().default('default value')

// Nullable
z.string().nullable()

// Strict object (no unknown keys)
z.object({ name: z.string() }).strict()

// Passthrough (allow unknown keys)
z.object({ name: z.string() }).passthrough()
```

---

## class-validator (NestJS)

### Setup

```bash
npm install class-validator class-transformer
```

### Enable in main.ts

```typescript
// src/main.ts
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true, // Strip non-whitelisted properties
      forbidNonWhitelisted: true, // Throw on non-whitelisted
      transform: true, // Transform payloads to DTO instances
      transformOptions: {
        enableImplicitConversion: true,
      },
    }),
  );

  await app.listen(3000);
}
```

### DTO Definition

```typescript
// src/users/dto/create-user.dto.ts
import {
  IsEmail,
  IsString,
  MinLength,
  MaxLength,
  IsOptional,
  IsInt,
  Min,
  Max,
  Matches,
} from 'class-validator';
import { Transform } from 'class-transformer';

export class CreateUserDto {
  @IsEmail({}, { message: 'Invalid email format' })
  @Transform(({ value }) => value.toLowerCase())
  email: string;

  @IsString()
  @MinLength(8, { message: 'Password must be at least 8 characters' })
  @Matches(/[A-Z]/, { message: 'Password must contain uppercase letter' })
  @Matches(/[0-9]/, { message: 'Password must contain number' })
  password: string;

  @IsString()
  @MinLength(2)
  @MaxLength(100)
  @IsOptional()
  name?: string;

  @IsInt()
  @Min(18)
  @Max(120)
  @IsOptional()
  age?: number;
}

// src/users/dto/update-user.dto.ts
import { PartialType } from '@nestjs/mapped-types';
import { CreateUserDto } from './create-user.dto';

export class UpdateUserDto extends PartialType(CreateUserDto) {}
```

### Query DTO

```typescript
// src/common/dto/pagination.dto.ts
import { IsInt, Min, Max, IsOptional, IsEnum } from 'class-validator';
import { Type } from 'class-transformer';

export class PaginationDto {
  @IsInt()
  @Min(1)
  @IsOptional()
  @Type(() => Number)
  page?: number = 1;

  @IsInt()
  @Min(1)
  @Max(100)
  @IsOptional()
  @Type(() => Number)
  limit?: number = 10;
}

export class SortDto {
  @IsEnum(['createdAt', 'name', 'email'])
  @IsOptional()
  sortBy?: string = 'createdAt';

  @IsEnum(['asc', 'desc'])
  @IsOptional()
  order?: 'asc' | 'desc' = 'desc';
}
```

### Common class-validator Decorators

```typescript
// String
@IsString()
@IsNotEmpty()
@MinLength(2)
@MaxLength(100)
@Matches(/regex/)

// Number
@IsNumber()
@IsInt()
@Min(0)
@Max(100)
@IsPositive()

// Boolean
@IsBoolean()

// Date
@IsDate()
@IsDateString()

// Array
@IsArray()
@ArrayMinSize(1)
@ArrayMaxSize(10)
@ArrayUnique()

// Nested objects
@ValidateNested()
@Type(() => NestedDto)

// Conditional validation
@ValidateIf((o) => o.type === 'email')
@IsEmail()

// Custom validation
@Validate(CustomValidator)
```

---

## Comparison

| Feature | Zod | class-validator |
|---------|-----|-----------------|
| Type inference | Excellent | Good (with class-transformer) |
| Bundle size | Smaller | Larger |
| Learning curve | Lower | Higher |
| NestJS integration | Manual | Native |
| Runtime validation | Yes | Yes |
| Transforms | Built-in | Separate (class-transformer) |
| Best for | Express, Fastify | NestJS |

## Validation Best Practices

1. **Validate at boundaries** - API inputs, not internal functions
2. **Whitelist properties** - Strip unknown fields
3. **Transform early** - Normalize data (lowercase emails, trim strings)
4. **Descriptive messages** - Help clients understand errors
5. **Consistent format** - Same error structure across all endpoints
6. **Separate concerns** - Validation ≠ business logic
