# 03. NestJS Backend Skeleton

> Guide for the structure, configuration, and common utilities of a backend application based on NestJS 11.
> See alongside [04-database-setup](./04-database-setup.md), [05-queue-worker](./05-queue-worker.md), and [06-auth-security](./06-auth-security.md).

---

## 1. Dependencies

### package.json

```json
{
  "name": "@scope/backend",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "dev": "nest start --watch",
    "build": "nest build",
    "start": "node dist/main.js",
    "worker": "node dist/worker.js",
    "lint": "eslint src --max-warnings 0",
    "typecheck": "tsc --noEmit -p tsconfig.json",
    "test": "jest",
    "clean": "rm -rf dist *.tsbuildinfo",
    "prisma:generate": "prisma generate",
    "prisma:migrate": "prisma migrate dev"
  },
  "dependencies": {
    "@nestjs/bullmq": "^11.0.1",
    "@nestjs/common": "^11.0.5",
    "@nestjs/config": "^4.0.0",
    "@nestjs/core": "^11.0.5",
    "@nestjs/jwt": "^11.0.0",
    "@nestjs/mongoose": "^11.0.1",
    "@nestjs/passport": "^11.0.0",
    "@nestjs/platform-express": "^11.0.5",
    "@nestjs/schedule": "^5.0.0",
    "@nestjs/swagger": "^11.2.6",
    "@nestjs/throttler": "^6.4.0",
    "@prisma/adapter-pg": "^7.0.0",
    "@prisma/client": "^7.0.0",
    "@scope/shared": "workspace:*",
    "bullmq": "^5.34.8",
    "class-transformer": "^0.5.1",
    "class-validator": "^0.14.1",
    "cookie-parser": "^1.4.7",
    "google-auth-library": "^10.5.0",
    "ioredis": "^5.9.3",
    "mongoose": "^8.9.5",
    "passport": "^0.7.0",
    "passport-jwt": "^4.0.1",
    "pg": "^8.13.1",
    "reflect-metadata": "^0.2.2",
    "rxjs": "^7.8.1",
    "sharp": "^0.34.5",
    "swagger-ui-express": "^5.0.1",
    "uuid": "^13.0.0"
  },
  "devDependencies": {
    "@nestjs/cli": "^11.0.2",
    "@nestjs/testing": "^11.0.5",
    "@types/cookie-parser": "^1.4.10",
    "@types/express": "^5.0.0",
    "@types/node": "^24.0.0",
    "@types/passport-jwt": "^4.0.1",
    "@types/uuid": "^11.0.0",
    "@scope/eslint-config": "workspace:*",
    "@scope/tsconfig": "workspace:*",
    "eslint": "^9.18.0",
    "prisma": "^7.0.0",
    "typescript": "^5.7.3"
  }
}
```

### nest-cli.json

```json
{
  "$schema": "https://json.schemastore.org/nest-cli",
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "deleteOutDir": true,
    "builder": "tsc",
    "tsConfigPath": "tsconfig.build.json"
  }
}
```

### tsconfig.json

```json
{
  "extends": "@scope/tsconfig/nestjs.json",
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] },
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "test", "**/*spec.ts"]
}
```

### tsconfig.build.json

```json
{
  "extends": "./tsconfig.json",
  "exclude": ["node_modules", "dist", "test", "**/*spec.ts"]
}
```

---

## 2. Directory Structure

```
apps/backend/src/
├── config/
│   └── configuration.ts          # Centralized environment variable management
├── common/
│   ├── decorators/
│   │   ├── current-user.decorator.ts
│   │   └── roles.decorator.ts
│   ├── dto/
│   │   └── pagination.dto.ts
│   ├── filters/
│   │   └── http-exception.filter.ts
│   ├── interceptors/
│   │   └── response-transform.interceptor.ts
│   └── utils/
│       ├── encryption.util.ts
│       └── pagination.util.ts
├── database/
│   ├── postgresql/
│   │   ├── prisma.service.ts
│   │   └── prisma.module.ts
│   └── mongodb/
│       ├── mongodb.module.ts
│       └── schemas/              # Mongoose schemas
├── jobs/
│   └── jobs.module.ts            # BullMQ queue registration
├── modules/
│   ├── auth/                     # Auth module → see 06-auth-security.md
│   └── users/                    # Users module
├── app.module.ts                 # Main module (HTTP Server)
├── main.ts                       # Entry point (HTTP)
├── worker.module.ts              # Worker module (Queue processors)
└── worker.ts                     # Worker entry point
```

---

## 3. main.ts (Entry Point)

```typescript
import "reflect-metadata";
import { NestFactory } from "@nestjs/core";
import { ValidationPipe } from "@nestjs/common";
import cookieParser from "cookie-parser";
import { DocumentBuilder, SwaggerModule } from "@nestjs/swagger";
import { AppModule } from "./app.module";
import { HttpExceptionFilter } from "./common/filters/http-exception.filter";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Cookie parsing (for Refresh Tokens, etc.)
  app.use(cookieParser());

  // Global ValidationPipe: Automatic DTO validation
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true, // Automatically strip non-whitelisted properties
      forbidNonWhitelisted: true, // Throw 400 error on unknown properties
      transform: true, // Automatically transform payloads to DTO instances
    }),
  );

  // Global Exception Filter: Consistent error response formatting
  app.useGlobalFilters(new HttpExceptionFilter());

  // API Prefix
  const apiPrefix = process.env.API_PREFIX ?? "/api";
  app.setGlobalPrefix(apiPrefix);

  // CORS
  app.enableCors({
    origin: process.env.FRONTEND_URL ?? "http://localhost:15310",
    credentials: true,
  });

  // Swagger — Enabled only in development
  if (process.env.NODE_ENV !== "production") {
    const config = new DocumentBuilder()
      .setTitle("API Documentation")
      .setVersion("1.0")
      .addBearerAuth(
        {
          type: "http",
          scheme: "bearer",
          bearerFormat: "JWT",
        },
        "access-token",
      )
      .build();

    const document = SwaggerModule.createDocument(app, config);
    SwaggerModule.setup("docs", app, document, {
      swaggerOptions: {
        persistAuthorization: true,
        tagsSorter: "alpha",
        operationsSorter: "alpha",
      },
    });
  }

  const port = process.env.PORT ?? 15320;
  await app.listen(port);
}

bootstrap();
```

---

## 4. AppModule

```typescript
import { Module } from "@nestjs/common";
import { ConfigModule } from "@nestjs/config";
import { ThrottlerModule } from "@nestjs/throttler";
import configuration from "./config/configuration";
import { PrismaModule } from "./database/postgresql/prisma.module";
import { MongodbModule } from "./database/mongodb/mongodb.module";
import { AuthModule } from "./modules/auth/auth.module";
import { UsersModule } from "./modules/users/users.module";
import { JobsModule } from "./jobs/jobs.module";

@Module({
  imports: [
    // Environment variables — Global, supports multiple env files
    ConfigModule.forRoot({
      isGlobal: true,
      load: [configuration],
      envFilePath: [".env.local", ".env"],
    }),

    // Rate Limiting (100 requests per 60 seconds)
    ThrottlerModule.forRoot([{ ttl: 60000, limit: 100 }]),

    // Databases
    PrismaModule,
    MongodbModule,

    // Queues
    JobsModule,

    // Business Modules
    UsersModule,
    AuthModule,
    // ... Additional modules
  ],
})
export class AppModule {}
```

---

## 5. configuration.ts (Centralized Config)

```typescript
// src/config/configuration.ts
export default () => ({
  nodeEnv: process.env.NODE_ENV ?? "development",
  port: parseInt(process.env.PORT ?? "15320", 10),
  apiPrefix: process.env.API_PREFIX ?? "/api",
  frontendUrl: process.env.FRONTEND_URL ?? "http://localhost:15310",

  database: {
    postgresUrl:
      process.env.DATABASE_URL ??
      "postgresql://postgres:postgres@localhost:15330/dbname",
    mongoUri: process.env.MONGODB_URI ?? "mongodb://localhost:15340/dbname_raw",
  },

  redis: {
    url: process.env.REDIS_URL ?? "redis://localhost:15350",
    host: process.env.REDIS_HOST ?? "localhost",
    port: parseInt(process.env.REDIS_PORT ?? "15350", 10),
    password: process.env.REDIS_PASSWORD,
  },

  auth: {
    jwtPrivateKey: process.env.JWT_PRIVATE_KEY,
    jwtPublicKey: process.env.JWT_PUBLIC_KEY,
    jwtAccessExpiry: process.env.JWT_ACCESS_EXPIRY ?? "15m",
    jwtRefreshExpiry: process.env.JWT_REFRESH_EXPIRY ?? "7d",
    googleClientId: process.env.GOOGLE_CLIENT_ID ?? "",
    googleClientSecret: process.env.GOOGLE_CLIENT_SECRET ?? "",
    allowedDomains: (() => {
      const domains = (process.env.ALLOWED_DOMAIN ?? "example.com")
        .split(",")
        .map((d) => d.trim().toLowerCase())
        .filter(Boolean);
      return domains.length > 0 ? domains : ["example.com"];
    })(),
  },

  encryptionKey: process.env.ENCRYPTION_KEY ?? "",
});
```

**Pattern**: Access using nested keys, such as `ConfigService.get<string>('database.postgresUrl')`.

---

## 6. Common Utilities

### HttpExceptionFilter

Converts all exceptions into a consistent format:

```typescript
// src/common/filters/http-exception.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
} from "@nestjs/common";
import { Response } from "express";
import type { ApiError } from "@scope/shared";

@Catch()
export class HttpExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(HttpExceptionFilter.name);

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();

    let statusCode = HttpStatus.INTERNAL_SERVER_ERROR;
    let message = "Internal server error";
    let error = "Internal Server Error";

    if (exception instanceof HttpException) {
      statusCode = exception.getStatus();
      const res = exception.getResponse();
      if (typeof res === "string") {
        message = res;
      } else if (typeof res === "object" && res !== null) {
        const resObj = res as Record<string, unknown>;
        message = (resObj["message"] as string) ?? exception.message;
        error = (resObj["error"] as string) ?? exception.name;
      }
    } else if (exception instanceof Error) {
      this.logger.error(
        `Unhandled exception: ${exception.message}`,
        exception.stack,
      );
    }

    const errorResponse: ApiError = {
      success: false,
      statusCode,
      message,
      error,
      timestamp: new Date().toISOString(),
    };

    response.status(statusCode).json(errorResponse);
  }
}
```

### ResponseTransformInterceptor

Wraps successful responses in a `{ success, data, timestamp }` format:

```typescript
// src/common/interceptors/response-transform.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from "@nestjs/common";
import { Observable } from "rxjs";
import { map } from "rxjs/operators";
import type { ApiResponse } from "@scope/shared";

@Injectable()
export class ResponseTransformInterceptor<T> implements NestInterceptor<
  T,
  ApiResponse<T>
> {
  intercept(
    _context: ExecutionContext,
    next: CallHandler,
  ): Observable<ApiResponse<T>> {
    return next.handle().pipe(
      map((data) => ({
        success: true,
        data,
        timestamp: new Date().toISOString(),
      })),
    );
  }
}
```

### PaginationDto + Utils

```typescript
// src/common/dto/pagination.dto.ts
import { ApiPropertyOptional } from "@nestjs/swagger";
import { IsOptional, IsInt, Min, Max } from "class-validator";
import { Type } from "class-transformer";

export class PaginationDto {
  @ApiPropertyOptional({
    description: "Page number",
    example: 1,
    default: 1,
    minimum: 1,
  })
  @IsOptional()
  @IsInt()
  @Min(1)
  @Type(() => Number)
  page?: number = 1;

  @ApiPropertyOptional({
    description: "Items per page",
    example: 20,
    default: 20,
    minimum: 1,
    maximum: 100,
  })
  @IsOptional()
  @IsInt()
  @Min(1)
  @Max(100)
  @Type(() => Number)
  limit?: number = 20;
}
```

```typescript
// src/common/utils/pagination.util.ts
export interface PaginatedResult<T> {
  data: T[];
  meta: {
    total: number;
    page: number;
    limit: number;
    totalPages: number;
    hasNext: boolean;
    hasPrev: boolean;
  };
}

export function buildPaginatedResult<T>(
  data: T[],
  total: number,
  page: number,
  limit: number,
): PaginatedResult<T> {
  const totalPages = Math.ceil(total / limit);
  return {
    data,
    meta: {
      total,
      page,
      limit,
      totalPages,
      hasNext: page < totalPages,
      hasPrev: page > 1,
    },
  };
}
```

### Custom Decorators

```typescript
// src/common/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from "@nestjs/common";
import type { User } from "../../generated/prisma/client";

export const CurrentUser = createParamDecorator(
  (_data: unknown, ctx: ExecutionContext): User => {
    const request = ctx.switchToHttp().getRequest<{ user: User }>();
    return request.user;
  },
);
```

```typescript
// src/common/decorators/roles.decorator.ts
import { SetMetadata } from "@nestjs/common";
import type { UserRole } from "@scope/shared";

export const ROLES_KEY = "roles";
export const Roles = (...roles: UserRole[]) => SetMetadata(ROLES_KEY, roles);
```

---

## 7. Decorator Rules

A `!` assertion is required for **all decorated fields** in Mongoose schemas and class-validator DTOs:

```typescript
// Mongoose Schema
@Prop() name!: string;

// DTO
@IsString() token!: string;
```

Reason: For compatibility between NestJS decorators and `strictPropertyInitialization`.

---

## 8. Swagger Decorator Patterns

### 8.1 DTO `@ApiProperty` Rules

- Required fields: `@ApiProperty` / Optional fields: `@ApiPropertyOptional`
- Always include `description` and `example`
- Specify the enum option for `enum` types: `@ApiProperty({ enum: UserRole })`
- Decorator order: **Swagger → Validation → Transformation**
- Use `!` assertion on class fields (see Section 7)

```typescript
// src/modules/users/dto/create-user.dto.ts
import { ApiProperty, ApiPropertyOptional } from "@nestjs/swagger";
import { IsEmail, IsString, IsOptional, IsEnum } from "class-validator";
import type { UserRole } from "@scope/shared";

export class CreateUserDto {
  @ApiProperty({ description: "Email address", example: "user@example.com" })
  @IsEmail()
  email!: string;

  @ApiProperty({ description: "User name", example: "John Doe" })
  @IsString()
  name!: string;

  @ApiPropertyOptional({
    description: "Role",
    enum: ["admin", "editor", "viewer"],
    default: "viewer",
  })
  @IsOptional()
  @IsEnum(["admin", "editor", "viewer"])
  role?: UserRole;
}
```

### 8.2 Response Schema Classes

`ResponseTransformInterceptor` uses `{ success, data, timestamp }`, and `HttpExceptionFilter` uses `{ success, statusCode, message, error, timestamp }`. Explicit schema classes are needed for Swagger to recognize these wrapper structures.

```typescript
// src/common/swagger/schemas.ts
import { ApiProperty, ApiPropertyOptional } from "@nestjs/swagger";

/** HttpExceptionFilter error response schema — reflects ApiError from @scope/shared */
export class ApiErrorSchema {
  @ApiProperty({ example: false })
  success!: boolean;

  @ApiProperty({ example: 400 })
  statusCode!: number;

  @ApiProperty({ example: "Validation failed" })
  message!: string;

  @ApiProperty({ example: "Bad Request" })
  error!: string;

  @ApiProperty({ example: "2026-02-28T12:00:00.000Z" })
  timestamp!: string;
}

/** PaginatedResult.meta schema */
export class PaginationMetaSchema {
  @ApiProperty({ example: 150 })
  total!: number;

  @ApiProperty({ example: 1 })
  page!: number;

  @ApiProperty({ example: 20 })
  limit!: number;

  @ApiProperty({ example: 8 })
  totalPages!: number;

  @ApiProperty({ example: true })
  hasNext!: boolean;

  @ApiProperty({ example: false })
  hasPrev!: boolean;
}
```

Domain-specific response classes are defined in their respective modules:

```typescript
// src/modules/users/swagger/user-response.schema.ts
import { ApiProperty } from "@nestjs/swagger";
import { PaginationMetaSchema } from "../../../common/swagger/schemas";

/** successful response for GET /users/:id */
export class UserResponse {
  @ApiProperty({ example: true })
  success!: boolean;

  @ApiProperty({
    example: { id: "uuid", email: "user@example.com", name: "John Doe" },
  })
  data!: Record<string, unknown>;

  @ApiProperty({ example: "2026-02-28T12:00:00.000Z" })
  timestamp!: string;
}

/** paginated response for GET /users */
export class UserPaginatedResponse {
  @ApiProperty({ example: true })
  success!: boolean;

  @ApiProperty({ example: { data: [], meta: {} } })
  data!: Record<string, unknown>;

  @ApiProperty({ type: PaginationMetaSchema })
  meta!: PaginationMetaSchema;

  @ApiProperty({ example: "2026-02-28T12:00:00.000Z" })
  timestamp!: string;
}
```

### 8.3 Controller Swagger Decorator Pattern

Example of an Users domain CRUD controller:

```typescript
// src/modules/users/users.controller.ts
import {
  Controller,
  Get,
  Post,
  Param,
  Body,
  Query,
  UseGuards,
  HttpStatus,
} from "@nestjs/common";
import {
  ApiTags,
  ApiBearerAuth,
  ApiOperation,
  ApiParam,
  ApiResponse,
} from "@nestjs/swagger";
import { JwtAuthGuard } from "../auth/guards/jwt-auth.guard";
import { CreateUserDto } from "./dto/create-user.dto";
import { PaginationDto } from "../../common/dto/pagination.dto";
import { ApiErrorSchema } from "../../common/swagger/schemas";
import {
  UserResponse,
  UserPaginatedResponse,
} from "./swagger/user-response.schema";

@ApiTags("Users")
@ApiBearerAuth("access-token")
@UseGuards(JwtAuthGuard)
@Controller("users")
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  @ApiOperation({
    summary: "List Users",
    description: "Returns a paginated list of users.",
  })
  @ApiResponse({
    status: HttpStatus.OK,
    description: "Success",
    type: UserPaginatedResponse,
  })
  @ApiResponse({
    status: HttpStatus.UNAUTHORIZED,
    description: "Unauthorized",
    type: ApiErrorSchema,
  })
  findAll(@Query() query: PaginationDto) {
    return this.usersService.findAll(query);
  }

  @Get(":id")
  @ApiOperation({ summary: "Get User by ID" })
  @ApiParam({
    name: "id",
    description: "User UUID",
    example: "550e8400-e29b-41d4-a716-446655440000",
  })
  @ApiResponse({
    status: HttpStatus.OK,
    description: "Success",
    type: UserResponse,
  })
  @ApiResponse({
    status: HttpStatus.UNAUTHORIZED,
    description: "Unauthorized",
    type: ApiErrorSchema,
  })
  @ApiResponse({
    status: HttpStatus.NOT_FOUND,
    description: "Not Found",
    type: ApiErrorSchema,
  })
  findOne(@Param("id") id: string) {
    return this.usersService.findById(id);
  }

  @Post()
  @ApiOperation({ summary: "Create User" })
  @ApiResponse({
    status: HttpStatus.CREATED,
    description: "Created",
    type: UserResponse,
  })
  @ApiResponse({
    status: HttpStatus.BAD_REQUEST,
    description: "Validation failed",
    type: ApiErrorSchema,
  })
  @ApiResponse({
    status: HttpStatus.UNAUTHORIZED,
    description: "Unauthorized",
    type: ApiErrorSchema,
  })
  @ApiResponse({
    status: HttpStatus.CONFLICT,
    description: "Email conflict",
    type: ApiErrorSchema,
  })
  create(@Body() dto: CreateUserDto) {
    return this.usersService.create(dto);
  }
}
```

### 8.4 Public Endpoint Pattern

Omit `@ApiBearerAuth` for endpoints that do not require authentication. The absence of the lock icon in Swagger UI intuitively marks these as public APIs.

```typescript
@ApiTags("Health")
@Controller("health")
export class HealthController {
  @Get()
  @ApiOperation({ summary: "Server health check" })
  @ApiResponse({ status: HttpStatus.OK, description: "Healthy" })
  check() {
    return { status: "ok" };
  }
}
```

### 8.5 Swagger Decorator Checklist

Ensure the following when writing all endpoints:

- [ ] `@ApiTags` — Set controller-level tags.
- [ ] `@ApiBearerAuth('access-token')` — Apply to controller or method level if authentication is required.
- [ ] `@ApiOperation({ summary })` — Add a description for each method.
- [ ] `@ApiParam` — Add descriptions for each path parameter (`/:id`).
- [ ] `@ApiResponse` — Define 200/201 success and all possible error status codes.
- [ ] Apply `@ApiProperty` or `@ApiPropertyOptional` to every DTO field.
- [ ] Ensure that `description` and `example` reflect realistic data.

---

## Verification

```bash
# Typecheck
pnpm --filter @scope/backend run typecheck

# ESLint
pnpm --filter @scope/backend run lint

# Build
pnpm --filter @scope/backend run build

# Test
pnpm --filter @scope/backend test
```
