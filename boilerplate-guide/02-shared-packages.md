# 02. Shared Packages

> Guide to configuring the three shared packages: `@scope/tsconfig`, `@scope/eslint-config`, and `@scope/shared`.
> They share consistent configurations and types across all apps in the monorepo.

---

## 1. @scope/tsconfig

A TypeScript configuration preset package. Each app inherits this using `extends`.

### package.json

```json
{
  "name": "@scope/tsconfig",
  "version": "0.0.1",
  "private": true,
  "files": ["*.json"]
}
```

### base.json

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "exclude": ["node_modules", "dist"]
}
```

### nestjs.json

For NestJS — Supports CommonJS modules and decorator metadata:

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "./base.json",
  "compilerOptions": {
    "module": "CommonJS",
    "moduleResolution": "node",
    "target": "ES2022",
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "isolatedModules": false,
    "outDir": "./dist",
    "baseUrl": "./"
  },
  "exclude": ["node_modules", "dist", "test", "**/*spec.ts"]
}
```

### nextjs.json

For Next.js — Supports DOM types, JSX, and App Router plugin:

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "./base.json",
  "compilerOptions": {
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "jsx": "preserve",
    "allowJs": true,
    "incremental": true,
    "noEmit": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "paths": {
      "@/*": ["./src/*"]
    },
    "plugins": [
      {
        "name": "next"
      }
    ]
  },
  "exclude": ["node_modules", ".next", "out"]
}
```

### Usage in Apps

```json
// apps/backend/tsconfig.json
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

```json
// apps/admin/tsconfig.json
{
  "extends": "@scope/tsconfig/nextjs.json",
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

---

## 2. @scope/eslint-config

ESLint 9 flat config preset. Provides 3 variants (base, nestjs, nextjs).

### package.json

```json
{
  "name": "@scope/eslint-config",
  "version": "0.0.1",
  "private": true,
  "files": ["*.mjs"],
  "exports": {
    "./base": "./base.mjs",
    "./nextjs": "./nextjs.mjs",
    "./nestjs": "./nestjs.mjs"
  },
  "peerDependencies": {
    "eslint": "^9.0.0"
  },
  "devDependencies": {
    "eslint": "^9.18.0",
    "@typescript-eslint/eslint-plugin": "^8.20.0",
    "@typescript-eslint/parser": "^8.20.0"
  }
}
```

### base.mjs

```javascript
import tseslint from "@typescript-eslint/eslint-plugin";
import tsParser from "@typescript-eslint/parser";

/** @type {import('eslint').Linter.Config[]} */
const config = [
  {
    files: ["**/*.{ts,tsx}"],
    plugins: {
      "@typescript-eslint": tseslint,
    },
    languageOptions: {
      parser: tsParser,
      parserOptions: {
        ecmaVersion: "latest",
        sourceType: "module",
      },
    },
    rules: {
      ...tseslint.configs.recommended.rules,
      "@typescript-eslint/no-unused-vars": [
        "error",
        { argsIgnorePattern: "^_" },
      ],
      "@typescript-eslint/no-explicit-any": "warn",
      "@typescript-eslint/explicit-function-return-type": "off",
      "@typescript-eslint/explicit-module-boundary-types": "off",
    },
  },
  {
    ignores: [
      "**/node_modules/**",
      "**/dist/**",
      "**/.next/**",
      "**/coverage/**",
    ],
  },
];

export default config;
```

### nestjs.mjs

```javascript
import baseConfig from "./base.mjs";

/** @type {import('eslint').Linter.Config[]} */
const config = [
  ...baseConfig,
  {
    files: ["**/*.ts"],
    rules: {
      "@typescript-eslint/interface-name-prefix": "off",
      "@typescript-eslint/explicit-function-return-type": "off",
      "@typescript-eslint/explicit-module-boundary-types": "off",
      "@typescript-eslint/no-explicit-any": "warn",
    },
  },
];

export default config;
```

### nextjs.mjs

```javascript
import baseConfig from "./base.mjs";

/** @type {import('eslint').Linter.Config[]} */
const config = [
  ...baseConfig,
  {
    files: ["**/*.{ts,tsx}"],
    rules: {
      "@typescript-eslint/no-require-imports": "off",
    },
  },
];

export default config;
```

### Usage in Apps

```javascript
// apps/backend/eslint.config.mjs
import nestjsConfig from '@scope/eslint-config/nestjs';
const config = [...nestjsConfig];
export default config;

// apps/admin/eslint.config.mjs
import nextjsConfig from '@scope/eslint-config/nextjs';
const config = [...nextjsConfig];
export default config;
```

---

## 3. @scope/shared

A package providing shared types, constants, and Zod validators between the backend and frontend.

### package.json

```json
{
  "name": "@scope/shared",
  "version": "0.0.1",
  "private": true,
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "require": "./dist/index.js",
      "types": "./dist/index.d.ts"
    }
  },
  "scripts": {
    "build": "tsc --project tsconfig.json",
    "typecheck": "tsc --noEmit",
    "clean": "rm -rf dist *.tsbuildinfo"
  },
  "devDependencies": {
    "@scope/tsconfig": "workspace:*",
    "typescript": "^5.7.3"
  },
  "dependencies": {
    "zod": "^3.24.1"
  }
}
```

**Key Point**: CJS build (`"main": "./dist/index.js"`) — NestJS relies on CommonJS, so `require()` compatibility is essential.

### tsconfig.json

```json
{
  "extends": "@scope/tsconfig/base.json",
  "compilerOptions": {
    "module": "CommonJS",
    "moduleResolution": "node",
    "outDir": "./dist",
    "rootDir": "./src",
    "composite": true,
    "isolatedModules": false
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### Directory Structure

```
packages/shared/src/
├── constants/
│   ├── limits.ts      # Pagination, rate-limit, and token expiration constants
│   ├── media.ts       # Media type constants
│   └── roles.ts       # User roles + hierarchy
├── types/
│   ├── api.types.ts   # API response/error interfaces
│   └── auth.types.ts  # Authentication related types
├── validators/
│   └── index.ts       # Zod schemas + inferred types
└── index.ts           # Barrel file
```

### Barrel File (index.ts)

```typescript
// Types
export * from "./types/api.types";
export * from "./types/auth.types";

// Constants
export * from "./constants/roles";
export * from "./constants/media";
export * from "./constants/limits";

// Validators
export * from "./validators";
```

### Type Example: API Response

```typescript
// src/types/api.types.ts
export interface PaginationParams {
  page?: number;
  limit?: number;
  sort?: string;
}

export interface PaginatedResponse<T> {
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

export interface ApiResponse<T> {
  success: boolean;
  data: T;
  message?: string;
  timestamp: string;
}

export interface ApiError {
  success: false;
  statusCode: number;
  message: string;
  error: string;
  timestamp: string;
}
```

### Type Example: Authentication

```typescript
// src/types/auth.types.ts
export type UserRole = "admin" | "editor" | "viewer";

export interface TokenPayload {
  sub: string;
  email: string;
  name: string;
  role: UserRole;
  iat?: number;
  exp?: number;
  iss?: string;
  aud?: string;
}

export interface AuthUser {
  id: string;
  googleId: string;
  email: string;
  name: string;
  profileImage: string | null;
  domain: string;
  role: UserRole;
  isActive: boolean;
  lastLoginAt: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface AuthTokens {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
}
```

### Constants Example

```typescript
// src/constants/roles.ts
import type { UserRole } from "../types/auth.types";

export const USER_ROLES = {
  ADMIN: "admin",
  EDITOR: "editor",
  VIEWER: "viewer",
} as const satisfies Record<string, UserRole>;

export const ROLE_HIERARCHY: Record<UserRole, number> = {
  admin: 3,
  editor: 2,
  viewer: 1,
};
```

```typescript
// src/constants/limits.ts
export const PAGINATION = {
  DEFAULT_PAGE: 1,
  DEFAULT_LIMIT: 20,
  MAX_LIMIT: 100,
} as const;

export const TOKEN_EXPIRY = {
  ACCESS: "15m",
  REFRESH: "7d",
} as const;

export const MAX_SESSIONS_PER_USER = 5;
```

### Zod Validator Pattern

```typescript
// src/validators/index.ts
import { z } from "zod";

export const paginationSchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().positive().max(100).default(20),
  sort: z.string().optional(),
});

// Auto-infer type from Zod schema
export type PaginationSchema = z.infer<typeof paginationSchema>;
```

**Pattern**: Every Zod schema must be accompanied by its `z.infer<typeof schema>` inferred type.

### Usage in Apps

```json
// apps/backend/package.json or apps/admin/package.json
{
  "dependencies": {
    "@scope/shared": "workspace:*"
  }
}
```

```typescript
// Usage example
import type { ApiResponse, UserRole } from "@scope/shared";
import { ROLE_HIERARCHY, paginationSchema } from "@scope/shared";
```

---

## Important Notes

- Changes in `@scope/shared` have a cascading impact across the monorepo.
- Ensure you verify all usages using `Grep` before modifying or deleting existing types.
- Always perform a full validation with `pnpm turbo typecheck` after any changes.

---

## Verification

```bash
# Build the shared package
pnpm --filter @scope/shared run build

# Run type check across the entire monorepo (verifies dependency integrity)
pnpm turbo typecheck

# Verify ESLint operation
pnpm turbo lint
```
