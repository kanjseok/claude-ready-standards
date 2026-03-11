# 04. Database Setup

> Dual database strategy guide using PostgreSQL (Prisma 7) + MongoDB (Mongoose 8).
> A pattern that separates relational domain data from unstructured raw data.

---

## 1. Dual DB Strategy Overview

| DB             | Purpose               | ORM/ODM    | Data Characteristics                                      |
| -------------- | --------------------- | ---------- | --------------------------------------------------------- |
| **PostgreSQL** | Domain data           | Prisma 7   | Normalization, Relations, ACID Transactions               |
| **MongoDB**    | Unstructured/Raw data | Mongoose 8 | Flexible schema, Heavy writes, Original data preservation |

**Usage Examples**:

- PostgreSQL: User, UserSession, Business entities, Metadata
- MongoDB: External API raw responses, Collection job logs, Large unstructured documents

---

## 2. PostgreSQL + Prisma 7

### schema.prisma

```prisma
generator client {
  provider        = "prisma-client"
  output          = "../src/generated/prisma"
  moduleFormat    = "cjs"
}

datasource db {
  provider = "postgresql"
}

// ─── Enums ───────────────────────────────────────────

enum UserRole {
  admin
  editor
  viewer
}

// ─── Models ──────────────────────────────────────────

model User {
  id           String    @id @default(uuid()) @db.Uuid
  googleId     String    @unique @map("google_id")
  email        String    @unique
  name         String
  profileImage String?   @map("profile_image") @db.Text
  role         UserRole  @default(viewer)
  isActive     Boolean   @default(true) @map("is_active")
  lastLoginAt  DateTime? @map("last_login_at") @db.Timestamptz()
  createdAt    DateTime  @default(now()) @map("created_at") @db.Timestamptz()
  updatedAt    DateTime  @updatedAt @map("updated_at") @db.Timestamptz()

  sessions UserSession[]

  @@map("users")
}

model UserSession {
  id               String    @id @default(uuid()) @db.Uuid
  userId           String    @map("user_id") @db.Uuid
  refreshTokenHash String    @map("refresh_token_hash")
  expiresAt        DateTime  @map("expires_at") @db.Timestamptz()
  revokedAt        DateTime? @map("revoked_at") @db.Timestamptz()
  ipAddress        String?   @map("ip_address") @db.VarChar(45)
  userAgent        String?   @map("user_agent") @db.Text
  createdAt        DateTime  @default(now()) @map("created_at") @db.Timestamptz()

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([refreshTokenHash])
  @@map("user_sessions")
}
```

**Prisma 7 Rules**:

- `generator client` → `provider: "prisma-client"` (Prisma 7 new syntax).
- `moduleFormat: "cjs"` — NestJS CommonJS compatibility.
- `output` → `src/generated/prisma/` (add to `.gitignore`).
- Use `@@map()` to explicitly name DB tables in snake_case.
- Use `@map()` to explicitly name DB columns in snake_case.

### PrismaService

```typescript
// src/database/postgresql/prisma.service.ts
import { Injectable, OnModuleDestroy, OnModuleInit } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import { PrismaPg } from "@prisma/adapter-pg";
import { PrismaClient } from "../../generated/prisma/client";

@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnModuleDestroy
{
  constructor(configService: ConfigService) {
    const adapter = new PrismaPg({
      connectionString: configService.get<string>("database.postgresUrl") ?? "",
    });
    super({ adapter });
  }

  async onModuleInit(): Promise<void> {
    await this.$connect();
  }

  async onModuleDestroy(): Promise<void> {
    await this.$disconnect();
  }
}
```

**Key Point**: Prisma 7 connects via the driver adapter method through `@prisma/adapter-pg`.

### PrismaModule (Global)

```typescript
// src/database/postgresql/prisma.module.ts
import { Global, Module } from "@nestjs/common";
import { PrismaService } from "./prisma.service";

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

The `@Global()` decorator permits injection of `PrismaService` into all modules unconditionally.

### Migration Workflow

```bash
# 1. Generate client (after schema changes)
pnpm --filter @scope/backend run prisma:generate

# 2. Create + apply migration files
pnpm --filter @scope/backend run prisma:migrate

# 3. Create SQL file only (for review)
pnpm --filter @scope/backend run prisma:migrate -- --create-only

# 4. Revalidate types
pnpm --filter @scope/backend run typecheck
```

### Import Pattern

```typescript
import type { User } from "../../generated/prisma/client";
import { PrismaService } from "../../database/postgresql/prisma.service";
```

---

## 3. MongoDB + Mongoose 8

### MongodbModule

```typescript
// src/database/mongodb/mongodb.module.ts
import { Module } from "@nestjs/common";
import { MongooseModule } from "@nestjs/mongoose";
import { ConfigModule, ConfigService } from "@nestjs/config";
import { RawData, RawDataSchema, Job, JobSchema } from "./schemas";

@Module({
  imports: [
    // Connection Setup
    MongooseModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService) => ({
        uri: configService.get<string>("database.mongoUri"),
      }),
      inject: [ConfigService],
    }),

    // Schema Registration
    MongooseModule.forFeature([
      { name: RawData.name, schema: RawDataSchema },
      { name: Job.name, schema: JobSchema },
    ]),
  ],
  exports: [MongooseModule],
})
export class MongodbModule {}
```

### Schema Definition Pattern

```typescript
// src/database/mongodb/schemas/raw-data.schema.ts
import { Prop, Schema, SchemaFactory } from "@nestjs/mongoose";
import { HydratedDocument } from "mongoose";

export type RawDataDocument = HydratedDocument<RawData>;

@Schema({
  timestamps: true,
  collection: "raw_data",
})
export class RawData {
  @Prop({ required: true, index: true })
  externalId!: string;

  @Prop({ required: true })
  source!: string;

  @Prop({ type: Object })
  data!: Record<string, unknown>;

  @Prop()
  fetchedAt!: Date;
}

export const RawDataSchema = SchemaFactory.createForClass(RawData);

// Compound Index
RawDataSchema.index({ source: 1, externalId: 1 }, { unique: true });
```

### Re-registration Across Modules

**Important**: Modules utilizing `@InjectModel()` must re-register the respective schema in their `MongooseModule.forFeature()`.

```typescript
// src/modules/processing/processing.module.ts
@Module({
  imports: [
    MongooseModule.forFeature([
      { name: RawData.name, schema: RawDataSchema }, // Re-registration is mandatory!
    ]),
  ],
  providers: [ProcessingService],
})
export class ProcessingModule {}
```

---

## 4. Transaction Patterns

### Prisma Transactions

```typescript
await this.prisma.$transaction(async (tx) => {
  const user = await tx.user.update({
    where: { id: userId },
    data: { lastLoginAt: new Date() },
  });
  await tx.userSession.create({
    data: { userId: user.id, refreshTokenHash, expiresAt },
  });
});
```

### Mongoose Sessions

```typescript
const session = await this.model.startSession();
session.startTransaction();
try {
  await this.model.create([data], { session });
  await session.commitTransaction();
} catch (error) {
  await session.abortTransaction();
  throw error;
} finally {
  session.endSession();
}
```

---

## Verification

```bash
# Verify PostgreSQL Connection
pnpm --filter @scope/backend run prisma:generate
pnpm --filter @scope/backend run prisma:migrate

# Verify MongoDB Connection (Auto-checked when dev server starts)
pnpm dev:backend

# Full typecheck
pnpm --filter @scope/backend run typecheck
```
