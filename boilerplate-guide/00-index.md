# Boilerplate Tech Guide

> Guide for quickly bootstrapping a pnpm monorepo-based NestJS + FastAPI + Kotlin + Next.js + Flutter full-stack project.
> This guide contains no business logic, covering only technical infrastructure and development rules.

---

## Tech Stack Summary

| Area                   | Technology                             | Version                               |
| ---------------------- | -------------------------------------- | ------------------------------------- |
| **Monorepo**           | pnpm + Turborepo                       | pnpm 10.30.0, turbo 2.3.3+            |
| **Runtime**            | Node.js                                | 24.14.0 (LTS)                         |
| **Language**           | TypeScript                             | 5.7+                                  |
| **Backend (JS)**       | NestJS                                 | 11.1.14                               |
| **Backend (Python)**   | FastAPI                                | 0.133.1                               |
| **Backend (Kotlin)**   | Spring Boot                            | 4.0.3                                 |
| **ORM (Kotlin SQL)**   | Exposed                                | 1.0.0                                 |
| **ORM (SQL)**          | Prisma                                 | 7.4.1                                 |
| **ORM (Python SQL)**   | SQLAlchemy                             | 2.0 async                             |
| **ODM (NoSQL)**        | Mongoose                               | 8                                     |
| **ODM (Python NoSQL)** | Motor                                  | 3.7.x                                 |
| **Queue**              | BullMQ + Redis                         | BullMQ 5, Redis 7.4.x                 |
| **State Mgmt**         | Tanstack Query                         | 5.66.0+                               |
| **Admin**              | Next.js + React + Tailwind CSS         | Next 16.1.6, React 19, Tailwind 4.2.1 |
| **Consumer Web**       | Next.js + React + Tailwind + Zustand + Serwist | Next 16.1.6, Zustand 5.0.x, Serwist 9.5.x |
| **Client**             | Flutter (macOS Desktop)                | SDK 3.41+                             |
| **DB (Relational)**    | PostgreSQL                             | 17.2                                  |
| **DB (Document)**      | MongoDB                                | 8.0.x                                 |
| **Cache/Queue**        | Redis                                  | 7.4.x                                 |
| **Container**          | Docker + Docker Compose                | 28.0.x                                |
| **Testing**            | Jest, Vitest, flutter_test, Playwright | -                                     |
| **CI Tool**            | Claude Code                            | Opus 4.6                              |

---

## Prerequisites

```bash
# Required
node --version     # v24+
corepack enable    # Auto-enables pnpm 10.30.0
docker --version   # Docker Desktop or OrbStack
git --version

# Flutter (For client development)
flutter --version  # 3.41+

# Kotlin (For Kotlin backend development)
java --version       # 17+
```

---

## Quick Start (10 Steps)

```bash
# 1. Clone repository
git clone <repo-url> && cd <project-name>

# 2. Install dependencies
pnpm install

# 3. Start dev infrastructure (PostgreSQL + MongoDB + Redis)
pnpm docker:dev

# 4. Set up environment variables
cp .env.example apps/backend/.env

# 5. Generate RSA key pair (for JWT signing)
cd tooling/scripts && npx tsx generate-keys.ts && cd ../..

# 6. Generate Prisma client & run migrations
pnpm --filter @example/backend run prisma:generate
pnpm --filter @example/backend run prisma:migrate

# 7. Build shared packages
pnpm --filter @example/shared build

# 8. Typecheck everything
pnpm turbo typecheck

# 9. Start dev server
pnpm turbo dev          # All
# Or individually: pnpm dev:admin / pnpm dev:backend

# 10. Flutter client (Optional)
cd apps/client && flutter pub get && flutter run
```

---

## Port Allocation Strategy

To avoid port conflicts with existing services in the development environment, a **BASE_PORT is randomly assigned between 11000 and 19900**. Ports for other resources are granted by **incrementing the BASE_PORT by 10**.

Here is an example of port allocation where the randomly assigned BASE_PORT is `15310`:

| Service        | Formula          | Example (BASE_PORT=15310) | Description                     |
| -------------- | ---------------- | ------------------------- | ------------------------------- |
| Consumer Web   | `BASE_PORT - 10` | (BASE_PORT=15310) 15300 | Consumer mobile web (PWA)       |
| Next.js Admin  | `BASE_PORT`      | 15310                     | Admin web frontend              |
| NestJS Backend | `BASE_PORT + 10` | 15320                     | REST API server                 |
| PostgreSQL     | `BASE_PORT + 20` | 15330                     | Relational DB (Domain data)     |
| MongoDB        | `BASE_PORT + 30` | 15340                     | Document DB (Unstructured data) |
| Redis          | `BASE_PORT + 40` | 15350                     | Queue broker + Cache            |
| Kotlin Backend | `BASE_PORT + 50` | 15360                     | REST API server (Kotlin)        |

---

## Package Structure

```
project-root/
├── apps/
│   ├── admin/          → @example/admin    (Next.js 16, App Router)
│   ├── web/            → @example/web      (Next.js 16, PWA)
│   ├── backend/        → @example/backend  (NestJS 11, CommonJS)
│   ├── backend-kt/     → Kotlin Backend  (Spring Boot 4, Gradle)
│   └── client/         → Flutter         (macOS desktop, out of pnpm)
├── packages/
│   ├── tsconfig/       → @example/tsconfig (Shared TS config)
│   ├── eslint-config/  → @example/eslint-config (ESLint 9 flat config)
│   └── shared/         → @example/shared   (Types + Constants + Zod validators)
├── tooling/
│   └── scripts/        → DB migrations, RSA key creation, etc.
├── infra/
│   └── docker/         → Dockerfiles (admin, backend, worker)
├── keys/               → RSA key pairs (.gitignore)
├── docker-compose.yml  → Dev infrastructure (PG + Mongo + Redis)
├── docker-compose.prod.yml → Production full services
├── turbo.json
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── package.json
└── .env.example
```

---

## Document List

| #                                        | Document                   | Content                                         |
| ---------------------------------------- | -------------------------- | ----------------------------------------------- |
| [01](./01-monorepo-setup.md)             | Monorepo Setup             | pnpm + Turborepo init, root config files        |
| [02](./02-shared-packages.md)            | Shared Packages            | tsconfig, eslint-config, shared packages        |
| [03](./03-backend-nestjs.md)             | NestJS Backend             | Skeleton, module structure, common utilities    |
| [04](./04-database-setup.md)             | Database Setup             | PostgreSQL(Prisma) + MongoDB(Mongoose)          |
| [05](./05-queue-worker.md)               | Queue + Worker             | BullMQ, processor pattern, separate process     |
| [06](./06-auth-security.md)              | Auth/Security              | JWT RS256, Google OAuth, Guard chain            |
| [07](./07-admin-nextjs.md)               | Admin Frontend             | Next.js 16, Tailwind 4, API client              |
| [08](./08-flutter-desktop.md)            | Flutter Client             | macOS desktop, Riverpod, Freezed                |
| [09](./09-docker-infrastructure.md)      | Docker Infrastructure      | Compose dev/prod, multi-stage builds            |
| [10](./10-testing-strategy.md)           | Testing Strategy           | Jest, Vitest, flutter_test, TDD                 |
| [11](./11-claude-code-integration.md)    | Claude Code Integration    | CLAUDE.md, rules, agents, hooks                 |
| [12](./12-dev-workflow-git.md)           | Dev Workflow               | Git rules, env vars, daily commands             |
| [13](./13-security-checklist.md)         | Security Checklist         | OWASP, secret management, security patterns     |
| [14](./14-backend-fastapi.md)            | FastAPI Backend            | Python skeleton, uv, SQLAlchemy 2.0, Motor      |
| [15](./15-backend-kotlin.md)             | Kotlin Backend             | Spring Boot 4 skeleton, Exposed, utilities      |
| [16](./16-monorepo-model-refactoring.md) | Monorepo Model Refactoring | Domain models, renaming, generalization runbook |
| [17](./17-consumer-web-pwa.md)           | Consumer Web (PWA)         | Mobile shell, Zustand, Serwist, FCM, offline-first |
| [18](./18-websocket-realtime.md)         | WebSocket & Real-Time      | Socket.IO, native WS, STOMP, Redis Pub/Sub scaling |

---

## Language Policy

When writing documents and communicating across the entire project, the following rules apply:

1. **English use**: All documents, user communications, and code bases (variable names, comments, etc.).

---

## Writing Principles

1. **Exclude Business Logic** — Includes only User/UserSession as auth boilerplate.
2. **Copy-and-Paste Ready** — Includes actual configuration files as code blocks.
3. **Self-Contained** — Each document can be referenced independently.
4. **Includes Verification** — Provides verification commands at the end of each document.
5. **English** — Keep the main content, comments, and variables in English.
