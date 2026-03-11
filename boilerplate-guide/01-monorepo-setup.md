# 01. Monorepo Setup

> Monorepo initialization guide based on pnpm 10 + Turborepo 2.
> Includes root configuration files (tsconfig, prettier, npmrc, gitignore).

---

## 1. Prerequisites

```bash
node --version       # v24+
corepack enable      # Manage pnpm with corepack
pnpm --version       # 10.30.0
```

---

## 2. Root package.json

```json
{
  "name": "project-name",
  "private": true,
  "packageManager": "pnpm@10.30.0",
  "engines": {
    "node": ">=24.0.0"
  },
  "scripts": {
    "dev": "turbo run dev",
    "dev:admin": "turbo run dev --filter=@scope/admin",
    "dev:backend": "turbo run dev --filter=@scope/backend",
    "dev:client": "cd apps/client && flutter run",
    "build": "turbo run build",
    "lint": "turbo run lint",
    "test": "turbo run test",
    "typecheck": "turbo run typecheck",
    "format": "prettier --write \"**/*.{ts,tsx,js,jsx,json,md}\"",
    "clean": "turbo run clean && rm -rf node_modules",
    "db:migrate": "turbo run db:migrate --filter=@scope/backend",
    "db:seed": "turbo run db:seed --filter=@scope/backend",
    "docker:dev": "docker compose up -d",
    "docker:down": "docker compose down"
  },
  "devDependencies": {
    "turbo": "^2.3.3",
    "prettier": "^3.4.2",
    "typescript": "^5.7.3"
  }
}
```

**Key Points**:

- Lock pnpm version using the `packageManager` field — automatically enabled by corepack.
- Enforce the minimum Node.js version using `engines.node`.
- Integrate the monorepo build/lint/test processes with Turborepo scripts.

---

## 3. pnpm-workspace.yaml

```yaml
packages:
  - apps/*
  - packages/*
  - tooling/*

ignoredBuiltDependencies:
  - "@scarf/scarf"

onlyBuiltDependencies:
  - "@nestjs/core"
  - "@prisma/engines"
  - msgpackr-extract
  - prisma
  - sharp
```

**Key Points**:

- Defines 3 workspace areas: `apps/*`, `packages/*`, and `tooling/*`.
- `onlyBuiltDependencies`: Only packages that require native addons are allowed to build (improves installation speed).
- `ignoredBuiltDependencies`: Ignores unnecessary build scripts, such as telemetry.

---

## 4. turbo.json

```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["tsconfig.base.json"],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**"],
      "env": ["NODE_ENV", "NEXT_PUBLIC_*"]
    },
    "dev": {
      "dependsOn": ["^build"],
      "cache": false,
      "persistent": true
    },
    "lint": {
      "dependsOn": ["^build"]
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"]
    },
    "typecheck": {
      "dependsOn": ["^build"]
    },
    "clean": {
      "cache": false
    },
    "db:migrate": {
      "cache": false
    },
    "db:seed": {
      "cache": false
    }
  }
}
```

**Key Points**:

- `^build`: Builds dependency packages (e.g., `@scope/shared`) first.
- `dev` task: Supports watch mode with `cache: false` + `persistent: true`.
- `globalDependencies`: Invalidates the entire cache whenever `tsconfig.base.json` changes.

---

## 5. tsconfig.base.json

Root TypeScript configuration inherited by all packages:

```json
{
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
    "sourceMap": true,
    "composite": true
  },
  "exclude": ["node_modules", "dist", ".next", "coverage"]
}
```

---

## 6. .prettierrc

```json
{
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "semi": true,
  "plugins": []
}
```

---

## 7. .npmrc

```ini
auto-install-peers=true
strict-peer-dependencies=false
```

---

## 8. .gitignore

```gitignore
# Dependencies
node_modules/
.pnpm-store/

# Build outputs
dist/
.next/
.turbo/
out/
build/

# Prisma generated client
apps/backend/src/generated/

# Environment
.env
.env.local
.env.*.local

# Logs
*.log
npm-debug.log*
pnpm-debug.log*

# Coverage
coverage/

# Editor
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Credentials
credentials/
*.pem
*.key
service-account*.json
client_secret*.json

# TypeScript
*.tsbuildinfo

# Flutter/Dart
.dart_tool/
.flutter-plugins-dependencies
.pub-cache/
.pub/
apps/client/build/
apps/client/macos/Pods/
apps/client/macos/Flutter/ephemeral/
*.iml
```

---

## 9. Directory Initialization Order

```bash
# 1. Initialize project
mkdir project-name && cd project-name
git init

# 2. Create root configuration files
# package.json, pnpm-workspace.yaml, turbo.json,
# tsconfig.base.json, .prettierrc, .npmrc, .gitignore

# 3. Create workspace directory structure
mkdir -p apps/{admin,backend,client}
mkdir -p packages/{tsconfig,eslint-config,shared}
mkdir -p tooling/scripts
mkdir -p infra/docker
mkdir -p keys

# 4. Install dependencies
pnpm install

# 5. Verify full build
pnpm turbo build
```

---

## Verification

```bash
# Check pnpm workspace recognition
pnpm ls --depth 0 -r

# Check Turborepo task graph
pnpm turbo run build --dry-run

# Typecheck
pnpm turbo typecheck
```
