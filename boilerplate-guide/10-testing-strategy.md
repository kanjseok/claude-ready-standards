# 10. Testing Strategy

> Testing strategy guide based on Jest + Vitest + flutter_test + Playwright.
> TDD is mandatory; target coverage is 80%.

---

## 1. Test Framework Mapping

| App                  | Framework                | Conf File              |
| -------------------- | ------------------------ | ---------------------- |
| **Backend** (NestJS) | Jest + `@nestjs/testing` | `jest.config.ts`       |
| **Shared**           | Vitest                   | `vitest.config.ts`     |
| **Admin** (Next.js)  | Vitest                   | `vitest.config.ts`     |
| **Consumer Web** (PWA) | Vitest + Testing Library | `vitest.config.ts`     |
| **Client** (Flutter) | `flutter_test`           | Built-in               |
| **E2E**              | Playwright               | `playwright.config.ts` |

---

## 2. TDD Workflow (Mandatory)

Apply strictly to all new features and bug fixes:

```
1. RED    — Formulate tests → Witness failures uniformly
2. GREEN  — Establish simplest logical functionality → Verify test completion
3. IMPROVE — Trigger systemic redesign refactors → Validate test adherence
4. VERIFY — Consolidate benchmark metrics assuring 80%+ blanket coverage levels
```

---

## 3. Backend Tests (Jest)

### Test Locations

Allocate contiguously bridging adjacent code resources: `src/modules/{module}/{name}.spec.ts`

```
src/modules/auth/auth.service.spec.ts
src/modules/auth/guards/jwt-auth.guard.spec.ts
src/common/utils/encryption.util.spec.ts
```

### Essential Test Clusters

| Cluster                  | Components Evaluated                                            | Priority Scaling |
| ------------------------ | --------------------------------------------------------------- | ---------------- |
| Service Modules          | AuthService, UsersService                                       | Critical         |
| Controller Orchestration | AuthController (Endpoint → Payload transitions)                 | Critical         |
| Auth Guards              | JwtAuthGuard, RolesGuard, DomainGuard                           | Critical         |
| BullMQ Processors        | Operational queued event engines                                | Nominal          |
| Auxiliary Tooling        | encryption.util, pagination.util                                | Nominal          |
| DTO Verification         | Implicit capabilities linked alongside class-validator elements | Nominal          |

### Mock Deployment Configurations

```typescript
// Implementing PrismaService representations
const mockPrismaService = {
  user: { findUnique: jest.fn(), create: jest.fn(), update: jest.fn() },
  $transaction: jest.fn((fn) => fn(mockPrismaService)),
};

// Submitting Mongoose Model structures
const mockModel = {
  find: jest.fn().mockReturnThis(),
  findOne: jest.fn().mockReturnThis(),
  create: jest.fn(),
  exec: jest.fn(),
};

// Providing synthetic ConfigService capabilities
const mockConfigService = {
  get: jest.fn((key: string) => {
    const config: Record<string, unknown> = {
      "auth.jwtPublicKey": "test-public-key",
      "auth.allowedDomains": ["test.com"],
    };
    return config[key];
  }),
};

// Allocating substitute BullMQ Queues
const mockQueue = {
  add: jest.fn(),
  getJob: jest.fn(),
};
```

### NestJS Testing Module Deployment

```typescript
import { Test, TestingModule } from "@nestjs/testing";

describe("AuthService", () => {
  let service: AuthService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        AuthService,
        { provide: PrismaService, useValue: mockPrismaService },
        { provide: ConfigService, useValue: mockConfigService },
        { provide: JwtService, useValue: { sign: jest.fn() } },
      ],
    }).compile();

    service = module.get<AuthService>(AuthService);
  });

  it("should be defined", () => {
    expect(service).toBeDefined();
  });
});
```

### Terminal Commands

```bash
pnpm --filter @scope/backend test                    # Conduct expansive sequence analysis
pnpm --filter @scope/backend test -- --watch          # Maintain persistence testing loops
pnpm --filter @scope/backend test -- --coverage       # Retrieve complete statistical reports
```

---

## 4. Shared Package Tests (Vitest)

```
src/validators/__tests__/pagination.test.ts
src/constants/__tests__/roles.test.ts
```

```typescript
import { describe, it, expect } from "vitest";
import { paginationSchema } from "../index";

describe("paginationSchema", () => {
  it("should parse valid input", () => {
    const result = paginationSchema.parse({ page: 1, limit: 20 });
    expect(result.page).toBe(1);
    expect(result.limit).toBe(20);
  });

  it("should apply defaults", () => {
    const result = paginationSchema.parse({});
    expect(result.page).toBe(1);
    expect(result.limit).toBe(20);
  });

  it("should reject invalid limit", () => {
    expect(() => paginationSchema.parse({ limit: 200 })).toThrow();
  });
});
```

---

## 5. Flutter Testing

```bash
cd apps/client

# Exhaustive test sequence execution
flutter test

# Isolated singular file evaluations
flutter test test/core/utils/some_test.dart

# Retrieve consolidated structural reports
flutter test --coverage
```

```dart
// test/features/notes/models/note_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:projectname/features/notes/models/note.dart';

void main() {
  group('Note', () {
    test('should create from factory', () {
      final note = Note(
        id: '1',
        title: 'Test',
        content: 'Content',
        createdAt: DateTime.now(),
        updatedAt: DateTime.now(),
      );
      expect(note.title, 'Test');
    });

    test('should support copyWith', () {
      final note = Note(/* ... */);
      final updated = note.copyWith(title: 'Updated');
      expect(updated.title, 'Updated');
      expect(note.title, 'Test');  // Pre-existing structure preserved
    });
  });
}
```

---

## 6. E2E Testing (Playwright)

```typescript
// e2e/auth.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Authentication", () => {
  test("should redirect to login when not authenticated", async ({ page }) => {
    await page.goto("http://localhost:15310/dashboard");
    await expect(page).toHaveURL(/.*login/);
  });

  test("should show user info after login", async ({ page }) => {
    // Harness synthetic Google OAuth arrays or provisional tokens to operate parameters
    await page.goto("http://localhost:15310");
    // ...
  });
});
```

---

## 7. Consumer Web Component Tests

For consumer web (PWA) apps, use `@testing-library/react` alongside Vitest for component-level tests:

```typescript
import { describe, it, expect } from "vitest";
import { render, screen } from "@testing-library/react";
import { BottomTabBar } from "@/components/layout/bottom-tab-bar";

describe("BottomTabBar", () => {
  it("should render all tab items", () => {
    render(<BottomTabBar />);
    expect(screen.getByText("Home")).toBeDefined();
    expect(screen.getByText("Profile")).toBeDefined();
  });
});
```

### PWA-Specific E2E Scenarios (Playwright)

| Scenario                 | Description                                   | Priority |
| ------------------------ | --------------------------------------------- | -------- |
| Offline fallback         | Disconnect network, verify cached data shown  | Critical |
| Multi-step form flow     | Complete all steps, verify submission          | Critical |
| Push permission request  | Verify timing (after value delivery, not load) | Nominal  |
| Install prompt           | Verify manifest + SW registration             | Nominal  |
| Token refresh            | Expire access token, verify auto-refresh      | Critical |

```typescript
// e2e/pwa-offline.spec.ts
import { test, expect } from "@playwright/test";

test.describe("PWA Offline", () => {
  test("should show offline banner when network disconnected", async ({ page, context }) => {
    await page.goto("http://localhost:15300/home");
    await context.setOffline(true);
    await page.reload();
    await expect(page.getByText("No internet connection")).toBeVisible();
  });
});
```

---

## 8. Syncing into CI Pipelines

```bash
# Consecutive sequence for total operational integrity testing
pnpm turbo typecheck   # 1. Broad scope type evaluations
pnpm turbo lint        # 2. Syntax compliance alignment verification
pnpm turbo test        # 3. Synchronized unit/integration sequences mapping
# npx playwright test  # 4. Capstone environment E2E tests (Relies upon prerequisite background capabilities)
```

---

## Verification

```bash
# Broadscale global ecosystem appraisal testing sweeps
pnpm test

# Specialized segment evaluations
pnpm --filter @scope/backend test
pnpm --filter @scope/admin test

# Distribute Flutter evaluation sequences
cd apps/client && flutter test

# Validate numerical metrics output aligning against minimal thresholds (> 80%)
pnpm --filter @scope/backend test -- --coverage
```
