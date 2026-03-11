# 11. Claude Code Integration

> Comprehensive guide on maximizing productivity and maintaining architectural consistency using the `claude` CLI.
> Includes `CLAUDE.md` directives, custom rules, agent prompts, and history logging methodologies.

---

## 1. CLAUDE.md Role and Structure

The `CLAUDE.md` file serves as the core rulebook for the AI agent (Claude Code). Claude analyzes it before tackling any task to comprehend the project's foundation.

### Default Project Definition Example

```markdown
# Claude Ready Standards Project Guidelines

...
```

### Key Principles Inside CLAUDE.md

1. **Architecture Synopsis**: Briefly define the exact technologies orchestrating the stack (e.g., Next.js App Router, NestJS Express, Prisma).
2. **Implementation Imperatives**: Document rules the AI naturally stumbles on (e.g., `Always adopt relative paths within modules`, `Adhere strictly to TDD protocols`).
3. **Execution Instructions**: Provide explicit commands detailing build, formatting, and test execution sequences (e.g., `pnpm --filter @scope/backend run test`).

---

## 2. `.claude/rules` Directory (Micro-Rulesets)

Use targeted rules under `.claude/rules/` rather than polluting `CLAUDE.md` with granular specifics. Claude Code proactively identifies and extracts relevant rules when encountering similar tasks based on keywords.

### Example: `.claude/rules/nestjs-controller.md`

```markdown
# NestJS Controller Formulation Rules

- **Target**: Files located in `apps/backend/src/modules/**/*.controller.ts`
- **Keywords**: nestjs, controller, swagger, api, endpoint

## Guidelines

1. **Swagger Segregation**: Mandatory incorporation of `@ApiTags()`, `@ApiOperation()`, and `@ApiResponse()`.
2. **DTO Exploitation**: Mandatory integration of `class-validator` derived schemas.
3. **Guards Application**: Enforce explicit `@UseGuards(JwtAuthGuard, RolesGuard)` application upon access-restricted APIs.
4. **Standard Responses**: Wrap primitive returns invoking `ApiResponse` format patterns.
```

---

## 3. Agent Prompts Strategy

For intricate development tasks, instruct Claude with granular, step-by-step methodologies rather than utilizing vague holistic commands.

### Scenario: Implementing an Entity in the Monorepo

**[❌ Avoid]**

> "Implement a Board Post feature." (Prone to logic skipping and arbitrary file generation).

**[✅ Recommended]**

> "Implement a 'Post' feature executing the sequential criteria defined below. Address each prompt step sequentially and await my endorsement post-completion before initiating consecutive stages:
>
> 1. Amend `packages/shared/schema` by introducing and exporting a `Post` model schema.
> 2. Alter `apps/backend/prisma/schema.prisma` accommodating the respective schema integration, generate the subsequent migration framework, and update the Prisma Client.
> 3. Architect the pertinent Module, Controller, Service, and Spec entities targeting the `apps/backend`. Validate passing testing parameters."

---

## 4. Keeping Track of History (Implementation Log)

AI tends to forget the chronological context of complex code refactoring loops over extended timeframes. Preserve the exact evolutionary footprint manually or prompt the AI to record it into persistent markdown journals.

### Structure of `docs/implementation-log.md`

```markdown
## YYYY-MM-DD: Task Title

**Goal:** Abstract the Authentication layer into `@scope/auth`.
**Actions Initiated:**

- Decoupled `AuthModule` dependencies away from `apps/backend`.
- Transitioned utilities originating from `common/utils/encryption` towards `packages/shared`.
  **Pending Actions:**
- Rectify broken test vectors in the `UserService`.
- Mitigate sequential circular dependency alerts surfacing randomly near `RoleGuard`.
```

---

## 5. Utilizing Hooks for Workflow Automation

Use `claude` hooks to inject custom preprocessing formatting constraints or intercept faulty test suites pre-commit organically.

### Example: Force linting locally prior to committals

```json
// .claude.json
{
  "hooks": {
    "preCommit": "pnpm turbo lint && pnpm turbo test"
  }
}
```

---

## 6. Verification Checklist

When assigning tasks to Claude, use this checklist to prevent scope creep and logic corruption:

- [ ] Has the exact operational domain/path constraint been explicitly communicated to the AI?
- [ ] Were the relevant CLI commands appended to assure validation checkbacks?
- [ ] Has the agent successfully formulated baseline test architectures mirroring RED/GREEN parameters before constructing absolute algorithms?
- [ ] Have arbitrary variable modifications impacting unmodified operational files been strictly vetoed?
