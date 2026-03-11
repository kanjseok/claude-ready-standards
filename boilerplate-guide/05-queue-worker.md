# 05. Queue + Worker Process

> BullMQ + Redis-based asynchronous task processing architecture guide.
> Offloads heavy tasks through a separate Worker process detached from the API server.

---

## 1. Architecture Overview

```
┌──────────────┐     enqueue     ┌─────────────┐
│  API Server  │ ──────────────▶ │    Redis    │
│  (main.ts)   │                 │  (BullMQ)   │
└──────────────┘                 └──────┬──────┘
                                        │ dequeue
                                 ┌──────▼──────┐
                                 │   Worker    │
                                 │ (worker.ts) │
                                 │ Processor A │
                                 │ Processor B │
                                 │ Processor C │
                                 └──────────────┘
```

- **API Server** (`main.ts`): Processes HTTP requests, enqueue tasks.
- **Worker** (`worker.ts`): Independent process solely executing queue processors (No HTTP server).
- **Redis**: BullMQ broker.

---

## 2. Queue Name Constants

```typescript
// src/jobs/queues/queue-names.ts
export const QUEUE_NAMES = {
  COLLECTION: "collection",
  MEDIA_DOWNLOAD: "media-download",
  NORMALIZATION: "normalization",
} as const;

export type QueueName = (typeof QUEUE_NAMES)[keyof typeof QUEUE_NAMES];
```

---

## 3. JobsModule (Queue Registration)

```typescript
// src/jobs/jobs.module.ts
import { Module } from "@nestjs/common";
import { BullModule } from "@nestjs/bullmq";
import { ConfigModule, ConfigService } from "@nestjs/config";
import { QUEUE_NAMES } from "./queues/queue-names";

@Module({
  imports: [
    // Redis Connection Setup
    BullModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService) => {
        const redisUrl =
          configService.get<string>("redis.url") ?? "redis://localhost:15350";
        const url = new URL(redisUrl);
        return {
          connection: {
            host: url.hostname,
            port: parseInt(url.port || "15350", 10),
            password: url.password || undefined,
          },
        };
      },
      inject: [ConfigService],
    }),

    // Queue Registration
    BullModule.registerQueue(
      { name: QUEUE_NAMES.COLLECTION },
      { name: QUEUE_NAMES.MEDIA_DOWNLOAD },
      { name: QUEUE_NAMES.NORMALIZATION },
    ),
  ],
  exports: [BullModule],
})
export class JobsModule {}
```

**Important**: Modules utilizing the queue **must re-register** it using `BullModule.registerQueue()`:

```typescript
// Usage in another module
@Module({
  imports: [BullModule.registerQueue({ name: QUEUE_NAMES.COLLECTION })],
  providers: [SomeService],
})
export class SomeModule {}
```

---

## 4. Enqueuing Tasks (Producer)

```typescript
// Queueing a task inside a service
import { InjectQueue } from "@nestjs/bullmq";
import { Queue } from "bullmq";
import { QUEUE_NAMES } from "../../jobs/queues/queue-names";

@Injectable()
export class SomeService {
  constructor(
    @InjectQueue(QUEUE_NAMES.COLLECTION)
    private readonly collectionQueue: Queue,
  ) {}

  async triggerCollection(accountId: string): Promise<void> {
    await this.collectionQueue.add(
      "collect", // job name
      { accountId }, // payload
      {
        attempts: 3, // maximum retries
        backoff: {
          type: "exponential",
          delay: 5000, // start delay of 5 sec, exponential increase
        },
        removeOnComplete: { count: 100 },
        removeOnFail: { count: 50 },
      },
    );
  }
}
```

---

## 5. Processor Pattern (Consumer)

```typescript
// src/modules/some-feature/processors/some.processor.ts
import { Processor, WorkerHost } from "@nestjs/bullmq";
import { Logger } from "@nestjs/common";
import { Job } from "bullmq";
import { QUEUE_NAMES } from "../../../jobs/queues/queue-names";

interface SomeJobData {
  accountId: string;
}

@Processor(QUEUE_NAMES.COLLECTION)
export class SomeProcessor extends WorkerHost {
  private readonly logger = new Logger(SomeProcessor.name);

  async process(job: Job<SomeJobData>): Promise<void> {
    this.logger.log(`Processing job ${job.id}: ${job.name}`);

    try {
      const { accountId } = job.data;

      // Idempotency Check — Ascertain if task was already processed
      // ...

      // Process task
      // ...

      // Report progress
      await job.updateProgress(50);

      // Enqueue sequentially into subsequent pipeline (optional)
      // await this.nextQueue.add('next-step', { ... });

      await job.updateProgress(100);
    } catch (error) {
      this.logger.error(
        `Failed to process job ${job.id}: ${(error as Error).message}`,
        (error as Error).stack,
      );
      throw error; // BullMQ triggers automatic retries upon failure
    }
  }
}
```

**Processor Rules**:

- `@Processor(QUEUE_NAME)`: Bind with dedicated queue name.
- `extends WorkerHost`: Essential NestJS BullMQ processor base class.
- **Idempotency is mandatory**: Re-processing the exact identical job must yield no adverse side effects.
- `job.updateProgress()`: Useful for tasks exceeding usual lifecycle durations.
- Errors thrown implicitly notify BullMQ to trigger automatic retries (pending `attempts` configuration).

---

## 6. Worker Process (Isolated Entry)

### worker.ts

```typescript
// src/worker.ts
import "reflect-metadata";
import { NestFactory } from "@nestjs/core";
import { Logger } from "@nestjs/common";
import { WorkerModule } from "./worker.module";

const logger = new Logger("Worker");

async function bootstrap() {
  // Bootstrap only the NestJS App Context omitting the HTTP server functionality
  const app = await NestFactory.createApplicationContext(WorkerModule, {
    logger: ["log", "error", "warn", "debug"],
  });

  app.enableShutdownHooks();

  logger.log("Worker process started");

  // Graceful shutdown
  const shutdown = async (signal: string) => {
    logger.log(`Received ${signal}, shutting down gracefully...`);
    await app.close();
    logger.log("Worker process stopped");
    process.exit(0);
  };

  process.on("SIGTERM", () => shutdown("SIGTERM"));
  process.on("SIGINT", () => shutdown("SIGINT"));
}

bootstrap().catch((error) => {
  logger.error("Failed to start worker", error);
  process.exit(1);
});
```

**Key Point**: `NestFactory.createApplicationContext()` — Starts the DI container strictly stripped of the HTTP server.

### WorkerModule

```typescript
// src/worker.module.ts
import { Module } from "@nestjs/common";
import { ConfigModule } from "@nestjs/config";
import configuration from "./config/configuration";
import { PrismaModule } from "./database/postgresql/prisma.module";
import { MongodbModule } from "./database/mongodb/mongodb.module";
import { JobsModule } from "./jobs/jobs.module";
import { UsersModule } from "./modules/users/users.module";
// ... exclusively import modules holding specific processors

/**
 * Worker module: Retains merely specific services, DB configs + queue processors.
 * Excludes the HTTP server, Auth mechanisms, and ThrottlerModule.
 */
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      load: [configuration],
      envFilePath: [".env.local", ".env"],
    }),
    PrismaModule,
    MongodbModule,
    JobsModule,
    UsersModule,
    // ... modules possessing processors
  ],
})
export class WorkerModule {}
```

---

## 7. Deployment Pattern

```bash
# Start API Server
node dist/main.js

# Start Worker process (distinct background process)
node dist/worker.js
```

Deploying as discrete services via Docker:

```yaml
# docker-compose.prod.yml
backend:
  command: ["node", "dist/main.js"]

worker:
  command: ["node", "dist/worker.js"]
  deploy:
    replicas: 2 # Arbitrarily scale worker replicas to meet load
```

→ For further nuances on Docker deployments, refer to [09-docker-infrastructure](./09-docker-infrastructure.md).

---

## 8. ETL Pipeline Pattern

Enchain discrete queues to organize multi-stage data processing:

```
Queue A (Collection) → Processor A → Queue B (Normalization) → Processor B → Queue C (Post-processing) → Processor C
```

Subsequent pipelined workloads are initiated by enqueuing into the consecutive queue sequentially after successfully discharging the preceding chore.

---

## Verification

```bash
# Validate Redis accessibility
docker compose up -d redis
redis-cli -p 15350 ping   # Output must be PONG

# Validate Worker initialization
pnpm --filter @scope/backend run build
node apps/backend/dist/worker.js
# → Inspect for logging indication: "Worker process started"
```
