# 12. Dev Workflow & Git

> Defines the Git branching model, Conventional Commits protocol, code review standards, and overall development workflow.

---

## 1. Branching Strategy (Feature Branch Flow)

- **`main`**: The primary branch representing the stable production build. Direct pushes are strictly **prohibited**. All merges obligate preceding Pull Requests (PRs).
- **`develop`** (Optional depending on project size): Serving as an integration branch for stashing active features awaiting deployment cycles.
- **`feature/*`**: Assigned uniformly for formulating novel characteristics. Generated dynamically deriving from `main` (or `develop`). Ex: `feature/user-auth`
- **`fix/*`**: Designated exclusively for patching defects. Ex: `fix/login-crash`
- **`hotfix/*`**: Immediate remedy deployments targeting the production environment urgently.
- **`chore/*`**: Devoted to minor maintenance efforts circumventing code source (e.g., config changes, dependency bumps). Ex: `chore/update-pnpm`

---

## 2. Conventional Commits

Mandatory enforcement of conventional commits format. Enforced automatically by `commitlint` paired alongside `husky`.

### Format Structure

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Types Designation

- `feat`: Appending nascent features
- `fix`: Devising bug rectifications
- `docs`: Documentation modifications entirely
- `style`: Aesthetic syntactical adjustments (formatting, semicolons) omitting logical shifts
- `refactor`: Algorithm modifications retaining preexisting behavior (e.g., Variable renaming)
- `perf`: Logic augmenting structural performance speeds
- `test`: Test sequence additions or revisions
- `chore`: Auxiliary tooling/configuration build revisions
- `build`: Dependency and execution compiler revisions
- `ci`: CI configuration operations

### Practical Examples

```bash
# Correct Implementations
git commit -m "feat(auth): implement google oauth login"
git commit -m "fix(admin): resolve pagination overlap anomaly"
git commit -m "docs(readme): update installation requirements"

# Prohibited Actions
git commit -m "update"
git commit -m "fix bug"
```

---

## 3. Workflow Procedure

### Step 1: Branch Creation

```bash
git checkout main
git pull origin main
git checkout -b feature/implement-dashboard
```

### Step 2: Implementation & Validation (TDD)

Enact development whilst maintaining repetitive testing thresholds.

```bash
pnpm test
pnpm lint
pnpm typecheck
```

### Step 3: Committal Process

Ensure granular commit milestones rather than colossal singular commits.

```bash
git add .
pnpm run commit  # (Utilizes commitizen if installed, alternatively proceed with git commit adhering strictly to Conventional syntax)
```

### Step 4: Synchronizing the Remote Repository

```bash
git push origin feature/implement-dashboard
```

### Step 5: Pull Request Generation

Initiate PRs integrating comprehensive details capturing the extent of changes, contextual backstories, and checklist fulfillments.

---

## 4. Code Review (PR) Guidelines

- Assure minimum viable approvals dictate PR closure parameters (ordinarily 1 endorsement).
- Squash merges define the default repository merger functionality (condensing multiple minor commits uniformly into a cohesive semantic milestone within `main`).
- Reviewers emphasize assessing architectural stability, naming convention adherence, test case viability, and potential security loopholes implicitly alongside logic verification.

---

## 5. Security & Secret Management within Version Control

- **Never** commit sensitive credentials (`.env`, PEM files, API keys).
- Rely on `.env.example` templates referencing required structural parameters exclusively.
- Use tools such as `git-secrets` or `trufflehog` to orchestrate preemptive interception targeting accidental credential disclosures.
- Refer strictly to the corresponding `boilerplate-guide/13-security-checklist.md` for extended directives.

---

## Verification

- [ ] Branch naming follows the convention (`feature/*`, `fix/*`, `hotfix/*`, `chore/*`)
- [ ] `commitlint` and `husky` are installed and configured

```bash
# Verify commitlint config exists
cat commitlint.config.js || cat commitlint.config.ts

# Verify husky hooks are installed
ls .husky/commit-msg
```

- [ ] A test commit with an invalid message is rejected

```bash
# This should fail
echo "test" | npx commitlint
# This should pass
echo "feat(auth): add login endpoint" | npx commitlint
```

- [ ] `.env` and key files are listed in `.gitignore`

```bash
grep -E "\.env|keys/" .gitignore
```

- [ ] Squash merge is the default merge strategy on the repository (configure via GitHub repository settings > Pull Requests > Allow squash merging)
