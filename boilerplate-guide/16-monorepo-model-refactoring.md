# 16. Monorepo Model Refactoring Runbook

> Comprehensive guide on renaming or broadening specific domain models uniformly spanning across the contiguous monorepo.
> Validating the full sequence spanning `@scope/shared` → `Prisma` → `Backend` → `Admin` instances.

---

## 1. Overview

This runbook walks through a complete model rename across every layer of a monorepo. While the example uses `Template` → `WorkspaceTemplate`, the same phased approach applies to any domain model rename or broadening operation.

### Layers Impacted

A single model rename touches **four layers** in sequence. Skipping or reordering layers leads to broken imports and runtime errors.

1. **Shared Package** (`packages/shared/`) — Types, Enums, Constants
2. **Database Schema** (`prisma/`) — Tables, Migrations
3. **Backend** (`apps/backend/`) — Modules, Controllers, Services, DTOs
4. **Frontend** (`apps/admin/`) — Routes, Hooks, Components

### Example Scenario

The examples below demonstrate renaming the `Template` entity to `WorkspaceTemplate`.

### Entities Impacted

1. **Shared Package**: Core Types, Enums, Routing Constants
2. **Prisma Schema**: Database tables & mappings
3. **Backend**: NestJS Modules, Controllers, Services, DTOs
4. **Admin**: Next.js App Router folders, Interfaces, Hooks, Components

---

## Phase 1: Shared Package Refactoring (`packages/shared/`)

Commence systematically addressing underlying foundational resources first.

### 1-1. Define Fundamental Definitions

```typescript
// Before
export type TemplateType = "system" | "custom";

// After
export type WorkspaceTemplateType = "system" | "custom";
```

### 1-2. Generalizing Role Restrictions

```typescript
// Before
export const TEMPLATE_ROLES = ["admin", "editor"] as const;

// After
export const WORKSPACE_TEMPLATE_ROLES = ["admin", "editor"] as const;
```

### 1-3. Build & Type Assertions

```bash
# Initiate build logic guaranteeing zero residual discrepancies encompassing foundational elements
pnpm --filter @scope/shared run build
```

---

## Phase 2: Database Schema Refactoring (`apps/backend/prisma/`)

Revamp database definitions matching overarching abstractions.

### 2-1. `schema.prisma` Alteration

```prisma
// Before
model Template {
  id        String   @id @default(uuid())
  name      String
  type      String   // Map to TemplateType
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@map("templates")
}

// After
model WorkspaceTemplate {
  id        String   @id @default(uuid())
  name      String
  type      String   // Map to WorkspaceTemplateType
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@map("workspace_templates") // Ensuring physical DB table name adjustments proceed linearly
}
```

### 2-2. Migration Architecture Revisions

Generate isolated migration files preceding DB resets.

```bash
cd apps/backend
npx prisma migrate dev --name rename_template_to_workspace_template
```

**⚠️ Caution:** Upon engaging data preservations natively, intervene directly inside generated SQL sequences explicitly writing `ALTER TABLE templates RENAME TO workspace_templates;` rather than relying upon arbitrary `DROP` alongside `CREATE` outputs.

---

## Phase 3: Backend Implementation Refactoring (`apps/backend/`)

Execute adjustments conforming NestJS APIs toward reformed models.

### 3-1. Transposing Module Directories

```bash
cd apps/backend/src/modules
mv template workspace-template
```

### 3-2. Re-labeling Source File Dependencies

Renaming `template.service.ts` to `workspace-template.service.ts` sequentially inside target modules.

### 3-3. Restructuring NestJS Components

```typescript
// Before
@Injectable()
export class TemplateService {
  constructor(private readonly prisma: PrismaService) {}

  async findMany() {
    return this.prisma.template.findMany();
  }
}

// After
@Injectable()
export class WorkspaceTemplateService {
  constructor(private readonly prisma: PrismaService) {}

  async findMany() {
    return this.prisma.workspaceTemplate.findMany();
  }
}
```

### 3-4. Readjusting Controller DTO Signatures & Swagger Details

```typescript
// Before
@ApiTags("Templates")
@Controller("templates")
export class TemplateController {
  @Get()
  @ApiResponse({ type: TemplateResponseDto })
  async getTemplates() {}
}

// After
@ApiTags("Workspace Templates")
@Controller("workspace-templates")
export class WorkspaceTemplateController {
  @Get()
  @ApiResponse({ type: WorkspaceTemplateResponseDto })
  async getWorkspaceTemplates() {}
}
```

### 3-5. Validating `AppModule` Dependencies

Validate exact file location routing updates within `app.module.ts`.

---

## Phase 4: Admin Implementation Refactoring (`apps/admin/`)

Modify Next.js React client patterns.

### 4-1. Realigning Routing Folders

Adjust routing architectures adhering to the Next.js App Router ruleset.

```bash
cd apps/admin/src/app/(dashboard)
mv templates workspace-templates
```

### 4-2. Restructuring API Hooks (SWR/React Query)

```typescript
// Before
export function useTemplates() {
  return useSWR<Template[]>("/api/templates", fetcher);
}

// After
export function useWorkspaceTemplates() {
  return useSWR<WorkspaceTemplate[]>("/api/workspace-templates", fetcher);
}
```

### 4-3. Remodeling UI View Abstractions

Assess internal React prop configurations and map explicit generic typing variables reliably matching exact components.

```tsx
// Before
interface TemplateListProps {
  templates: Template[];
}

// After
interface WorkspaceTemplateListProps {
  workspaceTemplates: WorkspaceTemplate[];
}
```

---

## Phase 5: Test Execution and Quality Restitution (`tests/`)

Ascertain unit/e2e components execute without discrepancy.

### 5-1. Adjust Synthetic Mocks

```typescript
// Before
const mockTemplate = { id: "1", name: "Test" };

// After
const mockWorkspaceTemplate = { id: "1", name: "Test" };
```

### 5-2. Rectify Endpoint Checks

```typescript
// Before
await request(app.getHttpServer()).get("/api/templates").expect(200);

// After
await request(app.getHttpServer()).get("/api/workspace-templates").expect(200);
```

---

## Critical Fallback Points

### 1) Global Search Implementations

Ordinarily AI interfaces or text editors skip plural forms implicitly depending strictly on explicit strings:

- `Template` -> `WorkspaceTemplate`
- `template` -> `workspaceTemplate`
- `templates` -> `workspaceTemplates`
- `TEMPLATE` -> `WORKSPACE_TEMPLATE`
- `templates/` -> `workspace-templates/`
- `.template.ts` -> `.workspace-template.ts`

**Solution**: Absolutely enforce `grep -rn` global scans organizing results categorically prior to systematic replacement functions.

---

## Verification

Run consecutive sequences affirming code purity outputs post-restoration:

```bash
# 1. Compile Shared ecosystem
pnpm --filter @scope/shared run build

# 2. Resurrect Prisma representations
pnpm --filter @scope/backend run prisma:generate

# 3. Synchronize Backend Typing
pnpm --filter @scope/backend run typecheck

# 4. Global Typings Compilation
pnpm turbo typecheck

# 5. Monolithic Build Verification
pnpm turbo build

# 6. Execute Testing Regiments
pnpm --filter @scope/backend test

# 7. Assure Seed Data Idempotency
pnpm --filter @scope/backend run seed

# 8. Filter Relic References Post-Cleanup
grep -r "OldModelName" apps/backend/src apps/admin/src packages/shared/src
```

Unanimous component success + DB Seed Idempotency + Pristine grep scans = Committal Approved.
