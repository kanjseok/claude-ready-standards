# 07. Admin Frontend (Next.js)

> Admin web frontend guide based on Next.js 16 + React 19 + Tailwind CSS 4.
> Covers App Router, typedRoutes, and auto-token-refresh API client.

---

## 1. Dependencies

### package.json

```json
{
  "name": "@scope/admin",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "dev": "next dev --port 15310",
    "build": "next build",
    "start": "next start --port 15310",
    "lint": "eslint src --max-warnings 0",
    "typecheck": "tsc --noEmit",
    "clean": "rm -rf .next out *.tsbuildinfo"
  },
  "dependencies": {
    "@scope/shared": "workspace:*",
    "next": "^16.1.6",
    "react": "^19.2.0",
    "react-dom": "^19.2.0"
  },
  "devDependencies": {
    "@scope/eslint-config": "workspace:*",
    "@scope/tsconfig": "workspace:*",
    "@types/node": "^24.0.0",
    "@types/react": "^19.2.0",
    "@types/react-dom": "^19.2.0",
    "eslint": "^9.18.0",
    "typescript": "^5.7.3",
    "tailwindcss": "^4.2.1",
    "@tailwindcss/postcss": "^4.2.1",
    "postcss": "^8.5.1"
  }
}
```

---

## 2. Config Files

### next.config.ts

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  transpilePackages: ["@scope/shared"], // Transpile monorepo packages
  typedRoutes: true, // Type-safe routing
  images: {
    remotePatterns: [
      {
        protocol: "https",
        hostname: "storage.googleapis.com",
      },
    ],
  },
};

export default nextConfig;
```

### tsconfig.json

```json
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

### postcss.config.mjs (Tailwind CSS 4)

```javascript
const config = {
  plugins: {
    "@tailwindcss/postcss": {},
  },
};

export default config;
```

Tailwind CSS v4 is configured as a PostCSS plugin directly, omitting a standalone `tailwind.config.ts`.

### eslint.config.mjs

```javascript
import nextjsConfig from "@scope/eslint-config/nextjs";

/** @type {import('eslint').Linter.Config[]} */
const config = [...nextjsConfig];

export default config;
```

---

## 3. Directory Structure

```
apps/admin/src/
├── app/            # App Router (Pages, Layouts)
│   ├── layout.tsx
│   ├── page.tsx
│   ├── login/
│   │   └── page.tsx
│   └── (dashboard)/
│       ├── layout.tsx
│       └── ...
├── components/     # Reusable React components
├── hooks/          # Custom React hooks
└── lib/            # Utilities, API client
    └── api-client.ts
```

---

## 4. typedRoutes Pattern

Setting `typedRoutes: true` inside `next.config.ts` allows for utilizing type-safe routes:

```typescript
import { type Route } from 'next';
import Link from 'next/link';
import { useRouter } from 'next/navigation';

// Link component
<Link href={`/users/${id}` as Route}>User Detail</Link>

// Programmatic routing
const router = useRouter();
router.replace('/login' as never);
```

---

## 5. API Client (Auto Token Refresh)

```typescript
// src/lib/api-client.ts
const API_BASE = process.env.NEXT_PUBLIC_API_URL ?? "http://localhost:15320";

let accessToken: string | null = null;
let isRefreshing = false;
let refreshQueue: Array<(token: string | null) => void> = [];

export function setAccessToken(token: string | null) {
  accessToken = token;
}

export function getAccessToken(): string | null {
  return accessToken;
}

async function refreshAccessToken(): Promise<string | null> {
  if (isRefreshing) {
    // Already refreshing — append to queue
    return new Promise((resolve) => {
      refreshQueue.push(resolve);
    });
  }

  isRefreshing = true;

  try {
    const response = await fetch(`${API_BASE}/api/auth/refresh`, {
      method: "POST",
      credentials: "include", // Transmits the httpOnly cookie
    });

    if (!response.ok) {
      accessToken = null;
      refreshQueue.forEach((resolve) => resolve(null));
      return null;
    }

    const data = (await response.json()) as { data: { accessToken: string } };
    const newToken = data.data?.accessToken ?? null;
    accessToken = newToken;
    refreshQueue.forEach((resolve) => resolve(newToken));
    return newToken;
  } catch {
    accessToken = null;
    refreshQueue.forEach((resolve) => resolve(null));
    return null;
  } finally {
    isRefreshing = false;
    refreshQueue = [];
  }
}

export async function apiRequest<T>(
  path: string,
  options: RequestInit = {},
): Promise<T> {
  const headers: Record<string, string> = {
    "Content-Type": "application/json",
    ...(options.headers as Record<string, string>),
  };

  if (accessToken) {
    headers["Authorization"] = `Bearer ${accessToken}`;
  }

  const response = await fetch(`${API_BASE}${path}`, {
    ...options,
    headers,
    credentials: "include",
  });

  // 401 → Automatically request token refresh + retry
  if (response.status === 401) {
    const newToken = await refreshAccessToken();
    if (!newToken) {
      throw new ApiError(401, "Session expired. Please log in again.");
    }

    headers["Authorization"] = `Bearer ${newToken}`;
    const retryResponse = await fetch(`${API_BASE}${path}`, {
      ...options,
      headers,
      credentials: "include",
    });

    if (!retryResponse.ok) {
      const error = (await retryResponse.json().catch(() => ({}))) as Record<
        string,
        unknown
      >;
      throw new ApiError(
        retryResponse.status,
        (error["message"] as string) ?? retryResponse.statusText,
      );
    }

    const retryData = (await retryResponse.json()) as { data: T };
    return retryData.data;
  }

  if (!response.ok) {
    const error = (await response.json().catch(() => ({}))) as Record<
      string,
      unknown
    >;
    throw new ApiError(
      response.status,
      (error["message"] as string) ?? response.statusText,
    );
  }

  if (response.status === 204) {
    return undefined as T;
  }

  const data = (await response.json()) as { data: T };
  return data.data;
}

export class ApiError extends Error {
  constructor(
    public readonly status: number,
    message: string,
  ) {
    super(message);
    this.name = "ApiError";
  }
}
```

**Core Patterns**:

- 401 Responses implicitly trigger an automatic `/auth/refresh` request
- Simultaneous clustered 401s → Triggers a solitary refresh call, pausing the siblings into a wait queue
- Original request retries automatically following a successful refresh outcome
- Deficient refresh outcomes yield a Session Expiration Exception

---

## 6. Server/Client Components Boundaries

```
Server Component (Defaults)
├── Fetching Data (fetch, DB querying)
├── Producing Metadata
└── Proceeding Static Rendering

Client Component ('use client')
├── Maintaining Status (useState, useReducer)
├── Orchestrating Event Triggers (onClick, onChange)
├── Utilizing Browser API resources (localStorage, window)
└── Auth Lifecycle Supervision (tokens, login/logout mechanisms)
```

---

## Verification

```bash
# Typecheck
pnpm --filter @scope/admin run typecheck

# ESLint
pnpm --filter @scope/admin run lint

# Dev Server
pnpm dev:admin
# → http://localhost:15310

# Build
pnpm --filter @scope/admin run build
```
