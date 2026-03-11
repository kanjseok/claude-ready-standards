# 09. Docker Infrastructure

> Docker Compose development/production environment + Multi-stage Dockerfile guide.
> Covers the containerization of the Postgres + MongoDB + Redis infrastructure and application services.

---

## 1. Development Infrastructure (docker-compose.yml)

During development, strictly initialize the dependencies (databases and Redis) using Docker:

```yaml
name: projectname

services:
  postgres:
    image: postgres:17
    ports:
      - "15330:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=dbname
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  mongodb:
    image: mongo:8
    ports:
      - "15340:27017"
    volumes:
      - mongodb_data:/data/db
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "15350:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
  mongodb_data:
  redis_data:
```

**Key Bulletins**:

- Implements port ranges bounding `153xx` — Nullifies conflicts with generic host-oriented operational services
- Distributes a ubiquitous `healthcheck` constraint across each configuration — Ensure sturdy boot sequences
- Uses Named volumes — Fosters persistent data structures

```bash
# Commence
pnpm docker:dev       # equates to docker compose up -d

# Scrutinize conditions
docker compose ps

# Halt configuration
docker compose down

# Halt while completely destroying attached persistent volumes
docker compose down -v
```

---

## 2. Production (docker-compose.prod.yml)

Run all 6 operational containers across Docker exclusively:

```yaml
name: projectname

services:
  web:
    build:
      context: .
      dockerfile: infra/docker/web.Dockerfile
    ports:
      - "3100:3100"
    environment:
      - NEXT_PUBLIC_API_URL=http://localhost:4000
      - NEXT_PUBLIC_GOOGLE_CLIENT_ID=${NEXT_PUBLIC_GOOGLE_CLIENT_ID}
      - NEXT_PUBLIC_FCM_VAPID_KEY=${NEXT_PUBLIC_FCM_VAPID_KEY}
    depends_on:
      - backend
    restart: unless-stopped

  admin:
    build:
      context: .
      dockerfile: infra/docker/admin.Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_URL=http://localhost:4000
      - NEXT_PUBLIC_GOOGLE_CLIENT_ID=${NEXT_PUBLIC_GOOGLE_CLIENT_ID}
    depends_on:
      - backend
    restart: unless-stopped

  backend:
    build:
      context: .
      dockerfile: infra/docker/backend.Dockerfile
    ports:
      - "4000:4000"
    environment:
      - PORT=4000
      - FRONTEND_URL=http://admin:3000
      - DATABASE_URL=postgresql://postgres:postgres@postgres:5432/dbname
      - MONGODB_URI=mongodb://mongodb:27017/dbname_raw
      - REDIS_URL=redis://redis:6379
      - JWT_PRIVATE_KEY=${JWT_PRIVATE_KEY}
      - JWT_PUBLIC_KEY=${JWT_PUBLIC_KEY}
      - GOOGLE_CLIENT_ID=${GOOGLE_CLIENT_ID}
      - GOOGLE_CLIENT_SECRET=${GOOGLE_CLIENT_SECRET}
      - ALLOWED_DOMAIN=${ALLOWED_DOMAIN:-example.com}
    volumes:
      - ./credentials:/app/credentials:ro
    depends_on:
      - postgres
      - mongodb
      - redis
    restart: unless-stopped

  worker:
    build:
      context: .
      dockerfile: infra/docker/worker.Dockerfile
    environment:
      # Environment variables uniformly mirroring the backend composition
      - DATABASE_URL=postgresql://postgres:postgres@postgres:5432/dbname
      - MONGODB_URI=mongodb://mongodb:27017/dbname_raw
      - REDIS_URL=redis://redis:6379
      # ... Remainder variables appended here...
    volumes:
      - ./credentials:/app/credentials:ro
    depends_on:
      - backend
    restart: unless-stopped
    deploy:
      replicas: 2 # Arbitrarily inflate configurations contingent upon payload volumes

  postgres:
    image: postgres:17
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=dbname
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    restart: unless-stopped

  mongodb:
    image: mongo:8
    volumes:
      - mongodb_data:/data/db
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  postgres_data:
  mongodb_data:
  redis_data:
```

**Production Stipulations**:

- Mask distinct secrets behind `${ENV_VAR}` variables (Implicit hardcoding remains prohibited)
- Ensure all volatile directories maintain a `:ro` (read-only) permission status
- Nullify exposed port references applied to Worker services; leverage `deploy.replicas` protocols when adjusting scale
- Reconcile parameter alignments bridging parallel environmental data between backend and worker nodes

---

## 3. Multi-stage Dockerfiles

### Admin (Next.js)

```dockerfile
# infra/docker/admin.Dockerfile
FROM node:24-alpine AS base
RUN corepack enable && corepack prepare pnpm@10.30.0 --activate

# Install dependencies
FROM base AS deps
WORKDIR /app
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json .npmrc ./
COPY apps/admin/package.json apps/admin/
COPY packages/shared/package.json packages/shared/
COPY packages/tsconfig/package.json packages/tsconfig/
COPY packages/eslint-config/package.json packages/eslint-config/
RUN pnpm install --frozen-lockfile --filter=@scope/admin...

# Build
FROM base AS builder
WORKDIR /app
COPY --from=deps /app .
COPY tsconfig.base.json ./
COPY packages/ packages/
COPY apps/admin apps/admin
RUN pnpm --filter=@scope/shared build
RUN pnpm --filter=@scope/admin build

# Production image
FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup -g 1001 -S nodejs && adduser -S nextjs -u 1001

COPY --from=builder --chown=nextjs:nodejs /app/apps/admin/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/apps/admin/.next/static ./apps/admin/.next/static
COPY --from=builder --chown=nextjs:nodejs /app/apps/admin/public ./apps/admin/public

USER nextjs
EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "apps/admin/server.js"]
```

### Consumer Web (Next.js PWA)

```dockerfile
# infra/docker/web.Dockerfile
FROM node:24-alpine AS base
RUN corepack enable && corepack prepare pnpm@10.30.0 --activate

# Install dependencies
FROM base AS deps
WORKDIR /app
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json .npmrc ./
COPY apps/web/package.json apps/web/
COPY packages/shared/package.json packages/shared/
COPY packages/tsconfig/package.json packages/tsconfig/
COPY packages/eslint-config/package.json packages/eslint-config/
RUN pnpm install --frozen-lockfile --filter=@scope/web...

# Build
FROM base AS builder
WORKDIR /app
COPY --from=deps /app .
COPY tsconfig.base.json ./
COPY packages/ packages/
COPY apps/web apps/web
RUN pnpm --filter=@scope/shared build
RUN pnpm --filter=@scope/web build

# Production image
FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup -g 1001 -S nodejs && adduser -S nextjs -u 1001

COPY --from=builder --chown=nextjs:nodejs /app/apps/web/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/apps/web/.next/static ./apps/web/.next/static
COPY --from=builder --chown=nextjs:nodejs /app/apps/web/public ./apps/web/public

USER nextjs
EXPOSE 3100
ENV PORT=3100
ENV HOSTNAME="0.0.0.0"

CMD ["node", "apps/web/server.js"]
```

Follows the same multi-stage pattern as the Admin Dockerfile. Key differences:
- Filters `@scope/web...` instead of `@scope/admin...`
- Requires `output: "standalone"` in `next.config.ts` (already set in [17-consumer-web-pwa](./17-consumer-web-pwa.md))
- Exposes a separate port from the Admin frontend

### Backend (NestJS)

```dockerfile
# infra/docker/backend.Dockerfile
FROM node:24-alpine AS base
RUN corepack enable && corepack prepare pnpm@10.30.0 --activate

# Install dependencies
FROM base AS deps
WORKDIR /app
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json .npmrc ./
COPY apps/backend/package.json apps/backend/
COPY packages/shared/package.json packages/shared/
COPY packages/tsconfig/package.json packages/tsconfig/
COPY packages/eslint-config/package.json packages/eslint-config/
RUN pnpm install --frozen-lockfile --filter=@scope/backend...

# Build
FROM base AS builder
WORKDIR /app
COPY --from=deps /app .
COPY tsconfig.base.json ./
COPY packages/ packages/
COPY apps/backend apps/backend
RUN pnpm --filter=@scope/shared build && pnpm --filter=@scope/backend build

# Production image
FROM node:24-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup -g 1001 -S nodejs && adduser -S nestjs -u 1001

COPY --from=builder --chown=nestjs:nodejs /app/apps/backend/dist ./dist
COPY --from=builder --chown=nestjs:nodejs /app/node_modules ./node_modules

USER nestjs
EXPOSE 4000

CMD ["node", "dist/main.js"]
```

### Worker

```dockerfile
# infra/docker/worker.Dockerfile
FROM node:24-alpine AS base
RUN corepack enable && corepack prepare pnpm@10.30.0 --activate

# The deps + builder stages identically align opposite their backend.Dockerfile counterpart structures

# Production image
FROM node:24-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup -g 1001 -S nodejs && adduser -S worker -u 1001

COPY --from=builder --chown=worker:nodejs /app/apps/backend/dist ./dist
COPY --from=builder --chown=worker:nodejs /app/node_modules ./node_modules

USER worker

# Assign dedicated worker origin endpoint (Distinct from universal API components)
CMD ["node", "dist/worker.js"]
```

---

## 4. Dockerfile Checklists

- [x] **Multi-stage configuration build structures**: Employs `deps` → `builder` → `runner` progressions (Eliminates excess image bloat)
- [x] **Non-root personnel specifications**: Reclaims operation integrity solely under explicit runner stage profiles
- [x] **`--frozen-lockfile` parameters**: Aborts compiling activities pending disjoint lock files (Sustains build replication integrity)
- [x] **Minimized `COPY` dependencies**: Sole restriction of designated file operations (Culls build-time contexts)
- [x] **Antecedent Shared Build sequence constraints**: Prescribes rigid protocols structuring `@scope/shared` tasks → before resolving application integrations

---

## Verification

```bash
# Substantiate the stability encompassing the primary Compose framework
docker compose config > /dev/null
docker compose -f docker-compose.prod.yml config > /dev/null

# Stimulate Development infrastructure
pnpm docker:dev
docker compose ps    # Review uniform healthy service thresholds

# Appraise targeted Production artifact constructs
docker compose -f docker-compose.prod.yml build
```
