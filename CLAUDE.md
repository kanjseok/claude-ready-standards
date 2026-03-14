# CLAUDE.md — Claude Ready Standards

> This file contains the Claude Code rules when working within the claude-ready-standards repository itself.
> For configuring CLAUDE.md in actual projects, please refer to `boilerplate-guide/11-claude-code-integration.md`.

---

## 0. Top Priority Principles

- **Pure Documentation Repository**: This repository contains no source code, build artifacts, or executables. It exclusively manages Markdown documents and Git configuration files.
- **Claude Code Exclusive**: All automation and AI-assisted tasks are performed exclusively with Claude Code.
- **Language Rules**:
  - For writing documents classified under the `docs/` folder and communicating with users: Use **English**.
  - For major documents in the project root (e.g., `README.md`) and all source code work (variable names, comments, etc.): Use **English**.

---

## 1. Repository Structure

```
claude-ready-standards/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug-report.yml       # Bug report issue template
│   │   └── document-request.yml # New document request template
│   └── PULL_REQUEST_TEMPLATE.md # PR checklist template
├── .gitignore           # Exclude OS/Editor files
├── .gitattributes       # Newline consistency + Markdown diff config
├── LICENSE              # CC BY 4.0 license
├── CODE_OF_CONDUCT.md   # Community code of conduct
├── CONTRIBUTING.md      # Contribution guidelines
├── README.md            # Repository entry point — Overview, catalog, writing principles
├── CLAUDE.md            # This file — Repository-specific Claude Code rules
├── OPEN-SOURCE-STARTER-GUIDE.md  # Guide to bootstrap new open source projects with Claude Code
├── TECH-STACK.md        # Tech stack version reference (Single Source of Truth)
└── boilerplate-guide/   # Boilerplate tech guides (19 documents)
    ├── 00-index.md      # Guide index + Tech stack summary
    ├── 01-monorepo-setup.md
    ├── 02-shared-packages.md
    ├── 03-backend-nestjs.md
    ├── 04-database-setup.md
    ├── 05-queue-worker.md
    ├── 06-auth-security.md
    ├── 07-admin-nextjs.md
    ├── 08-flutter-desktop.md
    ├── 09-docker-infrastructure.md
    ├── 10-testing-strategy.md
    ├── 11-claude-code-integration.md
    ├── 12-dev-workflow-git.md
    ├── 13-security-checklist.md
    ├── 14-backend-fastapi.md
    ├── 15-backend-kotlin.md
    ├── 16-monorepo-model-refactoring.md
    ├── 17-consumer-web-pwa.md
    └── 18-websocket-realtime.md
```

---

## 2. Document Writing Rules

### File Names

- Use **kebab-case**: `api-design-guide.md`
- Documents within category directories must use **number prefixes**: `00-index.md`, `01-topic.md`

### Heading Format

- One H1 (`#`) per file, used as the document title.
- Use H2 (`##`) to separate major sections.
- Use H3 (`###`) and below to structure detailed content.

### Blockquote Summary

- Include a 1-2 line summary in a `>` blockquote at the top of every document.

### Code Blocks

- Must specify the language: ``bash`, ``typescript`, ````json`, etc.
- Include configuration files in their complete form (ready for copy-pasting).

### Verification Section

- Place a `## Verification` section at the end of each document, including commands or checklists to confirm the setup setup is complete.

---

## 3. Git Rules

### Allowed Commands

```bash
git status
git diff [--cached] [--stat] [file]
git log [--oneline] [-n N]
git add <specific-files>     # Must specify files
git commit -m "..."
git push                     # Only when explicitly requested
```

### Prohibited Actions

```bash
git add -A              # Full staging prohibited
git add .               # Full staging prohibited
git push --force        # Force push prohibited
git reset --hard        # Hard reset prohibited
```

### Commit Message Format

Since this repository only handles documents, use the `docs` type as the default:

```
docs(<scope>): <subject>

## Task Details
[Detailed information]
```

**Scope Examples**:

- `tech-stack` — Changes to TECH-STACK.md
- `boilerplate` — Changes to boilerplate-guide documents
- `readme` — Changes to README.md
- `claude` — Changes to CLAUDE.md
- `repo` — Changes to repository configuration files (.gitignore, .gitattributes, etc.)

**Commit Example**:

```bash
git commit -m "docs(tech-stack): update Node.js version to 24

## Task Details
- Update version table in TECH-STACK.md
- Synchronize boilerplate-guide/00-index.md"
```

---

## 4. TECH-STACK.md Update Procedure

When versions change, strictly follow these 4 steps:

- [ ] **Step 1**: Research the latest stable versions for each technology.
- [ ] **Step 2**: Update the version table and `Last Updated` date in `TECH-STACK.md`.
- [ ] **Step 3**: Synchronize the tech stack summary table in `boilerplate-guide/00-index.md`.
- [ ] **Step 4**: Commit using the format `docs(tech-stack): update versions for YYYY-QN`.

---

## 5. Procedure for Adding New Standard Categories

When adding a new category (e.g., `api-design/`):

1. **Create Directory**: Create a kebab-case directory in the root.
2. **Create Index Document**: Write `<category>/00-index.md` (include blockquote summary and document list).
3. **Update README.md**: Add to the "Active Standards" table in the standard catalog.
4. **Update CLAUDE.md**: Reflect the change in the repository structure tree.
5. **Commit**: Commit using the format `docs(<category>): add initial standard documents`.

---

## 6. Work Workflow

### Default use of Plan Mode

- Must enter Plan Mode for tasks with 3+ steps or structural decisions.
- If issues occur during progress, stop immediately and re-plan — do not push through blindly.
- Utilize Plan Mode during the verification phase (plan the verification process, not just the writing).
- Write detailed specifications upfront to reduce ambiguity.

### Subagent Strategy

- Actively utilize subagents to keep the main context window clean.
- Delegate research, exploration, and parallel analysis to subagents.
- Deploy more computing resources via subagents for complex problems.
- Focus each subagent on a single task.

### Self-Improvement Loop

- Whenever you receive corrections from a user, document the pattern in `tasks/lessons.md`.
- Write your own rules to prevent repeating the same mistakes.
- Iteratively improve lessons until the error rate drops.
- Review lesson files for relevant projects at the start of a session.

### Verification Before Completion

- Do not mark a task as complete without proving it is correct.
- Compare changes with the main branch when relevant (`git diff`).
- Ask yourself: "Would a senior engineer approve this change?"
- Check cross-reference consistency between documents, link validity, and version table synchronization.

### Pursuing Quality (Balanced Approach)

- For non-trivial changes, ask yourself: "Is there a more elegant way?"
- If applying a patch-up fix, consider: "Implement an elegant solution considering all context so far."
- Avoid over-engineering for simple and obvious fixes.
- Critically review your own work before submitting.

### Autonomous Problem Solving

- When receiving a problem report, fix it directly — do not ask the user for step-by-step guidance.
- Find and resolve errors, inconsistencies, or omissions on your own.
- Minimize context-switching burdens on the user.
- If inconsistencies or errors between documents are found, fix them without separate instructions.

---

## 7. Task Management

1. **Plan Setup**: Write the plan as checkable items in `tasks/todo.md`.
2. **Plan Review**: Confirm with the user before starting work.
3. **Progress Tracking**: Check off completed items sequentially.
4. **Change Explanation**: Provide a high-level summary at each step.
5. **Document Results**: Add a review section in `tasks/todo.md`.
6. **Record Lessons**: Update `tasks/lessons.md` after corrections.

---

## 8. Core Principles

- **Simplicity First**: Keep all changes as simple as possible. Modify the minimum required documents.
- **Fix Root Causes**: No temporary fixes. Apply standards expected of a senior developer.
- **Minimal Impact**: Modify only what is necessary. Avoid causing side effects in other documents.
