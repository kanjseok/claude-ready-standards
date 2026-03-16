# Open Source Starter Guide for Claude Code

> A step-by-step guide for bootstrapping a new open source project with all essential structures, using Claude Code as the sole automation tool.
> Copy the prompts and checklists below to have Claude Code generate a production-ready open source repository from scratch.

---

## Purpose

Starting an open source project requires a surprising amount of boilerplate — license files, contribution guidelines, issue templates, CI configuration, and more. Most of this is repetitive and error-prone when done manually.

This guide provides **Claude Code-ready instructions** that generate all essential open source infrastructure in a single session. It covers:

- Repository initialization and Git configuration
- License selection and file generation
- Community health files (Code of Conduct, Contributing Guide, Security Policy)
- GitHub templates (Issues, Pull Requests, Discussions)
- `CLAUDE.md` setup for ongoing AI-assisted development
- README scaffolding
- `.gitignore` and `.gitattributes` configuration
- Optional extras (changelog, CI/CD, release automation)

---

## Prerequisites

- **Claude Code** installed and authenticated (`claude` CLI available)
- **Git** installed and configured with your identity
- A **GitHub account** (or compatible Git hosting platform)
- A clear idea of your project's **name**, **purpose**, and **primary language/stack**

---

## 1. Repository Initialization

### 1.1 Create the Project Directory

```bash
mkdir my-project && cd my-project
git init
```

### 1.2 Claude Code Prompt

> **Prompt to Claude Code:**
>
> "Initialize this repository as an open source project called `{project-name}`. The primary language is `{language/stack}`. Create the following foundational files:
>
> 1. `.gitignore` — tailored for `{language/stack}`, including OS files, editor files, and AI tool directories
> 2. `.gitattributes` — enforce LF line endings, set language-specific diff drivers
> 3. `README.md` — with project title, one-line description placeholder, badges placeholder, installation section, usage section, contributing link, and license section
> 4. `LICENSE` — using `{license-type}` (e.g., MIT, Apache-2.0, CC BY 4.0)
>
> Do not create any source code files yet."

---

## 2. License Selection Reference

Choose the appropriate license based on your project type:

| License | Best For | Permissions | Conditions |
|---------|----------|-------------|------------|
| **MIT** | Libraries, tools, general software | Commercial use, modification, distribution | Attribution required |
| **Apache-2.0** | Enterprise-friendly software | Commercial use, patent grant, modification | Attribution, state changes, include license |
| **GPL-3.0** | Projects requiring derivative openness | Commercial use, modification, distribution | Disclose source, same license for derivatives |
| **CC BY 4.0** | Documentation, creative works, standards | Commercial use, modification, distribution | Attribution required |
| **BSD-3-Clause** | Academic, research projects | Commercial use, modification, distribution | Attribution, no endorsement claims |
| **Unlicense** | Public domain dedication | Unrestricted | None |

### Claude Code Prompt for License

> "Generate a `LICENSE` file using the `{license-type}` license. Set the copyright year to `{year}` and the copyright holder to `{name-or-organization}`."

---

## 3. Community Health Files

Community health files are essential for encouraging contributions and maintaining project quality.

> **File Placement Note:** GitHub recognizes community health files (`CODE_OF_CONDUCT.md`, `CONTRIBUTING.md`, `SECURITY.md`) in three locations: the repository root, `docs/`, or `.github/`. This guide places them in the **root directory** for maximum visibility — contributors see them immediately without navigating into subdirectories. If you prefer a cleaner root, move them to `.github/` or `docs/` by updating the file paths in the generation prompts (Section 3) and the verification script (Section 12).

### 3.1 Code of Conduct

> **Prompt to Claude Code:**
>
> "Create a `CODE_OF_CONDUCT.md` based on the Contributor Covenant v2.1. Include sections for: Our Pledge, Our Standards (positive and negative examples), Enforcement Responsibilities, Scope, Reporting (via GitHub Issues and maintainer contact), and Attribution. Use `{security-contact-email}` as the reporting contact."

### 3.2 Contributing Guide

> **Prompt to Claude Code:**
>
> "Create a `CONTRIBUTING.md` with the following sections:
>
> 1. **Before You Start** — search existing issues, open issues for non-trivial changes, link to Code of Conduct
> 2. **How to Contribute** — fork & branch workflow with commands, branch naming conventions (`feature/`, `fix/`, `chore/`)
> 3. **Commit Message Convention** — Conventional Commits format with `{type}` as the default type (e.g., `feat` for software, `docs` for documentation), list of scopes, and examples of correct/incorrect messages
> 4. **Code/Document Writing Conventions** — `{project-specific-rules}`
> 5. **Pull Request Guidelines** — fill template, reference issues, keep PRs focused, cross-reference consistency
> 6. **Reporting Issues** — link to issue templates
> 7. **License** — state that contributions fall under the project license
>
> Include concrete command examples for the fork-and-branch workflow."

### 3.3 Security Policy

> **Prompt to Claude Code:**
>
> "Create a `SECURITY.md` with the following sections:
>
> 1. **Supported Versions** — table of versions receiving security updates
> 2. **Reporting a Vulnerability** — private disclosure instructions (email or GitHub Security Advisories), expected response timeline (e.g., 48 hours acknowledgment, 90 days fix)
> 3. **Disclosure Policy** — responsible disclosure process
> 4. **Security Update Process** — how patches are released and communicated
>
> Use `{security-contact-email}` as the contact."

---

## 4. GitHub Templates

### 4.1 Issue Templates

> **Prompt to Claude Code:**
>
> "Create GitHub issue templates under `.github/ISSUE_TEMPLATE/`:
>
> **`bug-report.yml`:**
> - Fields: Affected area (input, required), Issue type (dropdown with options: 'incorrect behavior', 'crash', 'performance', 'security', 'other'), Description (textarea, required), Steps to reproduce (textarea, required), Expected behavior (textarea), Environment info (textarea: OS, language version, project version)
>
> **`feature-request.yml`:**
> - Fields: Feature title (input, required), Motivation/problem (textarea, required), Proposed solution (textarea, required), Alternatives considered (textarea), Additional context (textarea)
>
> Use YAML form syntax (not classic Markdown templates). Add a `bug` label to `bug-report.yml` and an `enhancement` label to `feature-request.yml`."

### 4.2 Pull Request Template

> **Prompt to Claude Code:**
>
> "Create `.github/PULL_REQUEST_TEMPLATE.md` with the following structure:
>
> - **Summary** — 1-3 sentence description (comment placeholder)
> - **Related Issue** — link with `Closes #` syntax
> - **Changes** — bulleted list placeholder
> - **Checklist** — checkboxes for: follows coding conventions, tests pass, documentation updated, cross-references consistent, commit messages follow convention"

### 4.3 Discussion Templates (Optional)

> **Prompt to Claude Code:**
>
> "Create `.github/DISCUSSION_TEMPLATE/` with templates for:
>
> - **`ideas.yml`** — for feature brainstorming (title, description, use case fields)
> - **`q-and-a.yml`** — for questions (question field, context field, what you've tried field)"

---

## 5. CLAUDE.md Configuration

The `CLAUDE.md` file is the most critical piece for ongoing Claude Code integration. It defines how the AI agent understands and interacts with your project.

### 5.1 Structure Template

> **Prompt to Claude Code:**
>
> "Create a `CLAUDE.md` file for this project with the following sections:
>
> **Section 0 — Top Priority Principles:**
> - Project type and description in one sentence
> - Primary language/stack declaration
> - Language rules (what language for code, comments, docs, user communication)
> - Any absolute constraints (e.g., 'never modify files in vendor/')
>
> **Section 1 — Repository Structure:**
> - ASCII tree of the current directory structure
> - Brief description of each top-level directory and key file
>
> **Section 2 — Development Rules:**
> - File naming conventions
> - Code style requirements (formatting, linting tools)
> - Testing requirements (framework, coverage expectations, TDD preference)
> - Build and run commands
>
> **Section 3 — Git Rules:**
> - Allowed git commands (status, diff, log, add specific files, commit, push only when requested)
> - Prohibited git commands (under a `### Prohibited Actions` heading): `git add -A`, `git add .`, `git push --force`, `git reset --hard`
> - Commit message format with examples
> - Branch naming conventions
>
> **Section 4 — Workflow:**
> - Plan Mode requirements (when to enter, when to re-plan)
> - Subagent strategy (delegate research, single-task focus)
> - Verification before completion (prove correctness, check consistency)
> - Autonomous problem solving (fix directly, don't ask for step-by-step guidance)
>
> **Section 5 — Task Management:**
> - Plan in tasks/todo.md with checkable items
> - Confirm with user before starting
> - Track progress by checking off items
> - Record lessons in tasks/lessons.md
>
> **Section 6 — Core Principles:**
> - Simplicity first, fix root causes, minimal impact
>
> Tailor all examples and commands to `{language/stack}`."

### 5.2 Key Rules to Include

These rules prevent common mistakes when working with Claude Code:

```markdown
## Git Rules

### Allowed Commands
git status
git diff [--cached] [--stat] [file]
git log [--oneline] [-n N]
git add <specific-files>     # Must specify files explicitly
git commit -m "..."
git push                     # Only when explicitly requested

### Prohibited Actions
git add -A              # Full staging prohibited
git add .               # Full staging prohibited
git push --force        # Force push prohibited
git reset --hard        # Hard reset prohibited
```

### 5.3 The `.claude/rules/` Directory

For larger projects, supplement `CLAUDE.md` with granular rule files:

```text
.claude/
└── rules/
    ├── api-endpoints.md      # Rules for writing API endpoints
    ├── database-migrations.md # Rules for database schema changes
    ├── testing.md            # Test writing conventions
    └── security.md           # Security-sensitive code rules
```

> **Prompt to Claude Code:**
>
> "Create a `.claude/rules/` directory with a rule file for `{domain}`. The rule file should specify:
> - Target file patterns
> - Keywords for automatic matching
> - Numbered guidelines specific to `{domain}`
>
> Use the following format:
> ```markdown
> # {Domain} Rules
> - **Target**: `{glob pattern}`
> - **Keywords**: keyword1, keyword2, keyword3
>
> ## Guidelines
> 1. ...
> ```
> "

---

## 6. README Structure

A well-structured README is the front door of any open source project.

### Recommended Sections

```markdown
# Project Name

> One-line description of the project.

## Why {Project Name}?

- **Problem**: What pain point does this solve?
- **Solution**: How does this project address it?
- **Scope**: What does it cover?

## Quick Start

### Prerequisites
### Installation
### Basic Usage

## Documentation

Link to docs directory, wiki, or external documentation site.

## Contributing

Link to CONTRIBUTING.md.

## License

License type with link to LICENSE file.

## Author

Creator and maintainer information.
```

> **Prompt to Claude Code:**
>
> "Update the `README.md` for `{project-name}` with the structure above. The project is `{one-line-description}`. Fill in the Why section with: Problem — `{problem}`, Solution — `{solution}`, Scope — `{scope}`. Add placeholder sections for Quick Start, Documentation, Contributing, and License. Use `{license-type}` for the license section."

---

## 7. Git Configuration Files

### 7.1 `.gitignore`

Essential patterns to include regardless of stack:

```gitignore
# OS generated files
.DS_Store
Thumbs.db
Desktop.ini

# Editor / IDE
.vscode/
.idea/
*.code-workspace
*.swp
*.swo
*~

# AI tools
.claude/

# Environment files (NEVER commit secrets)
.env
.env.local
.env.*.local

# Task management (local only)
tasks/

# Dependencies (language-specific — add below)
```

> **Prompt to Claude Code:**
>
> "Create a `.gitignore` file. Start with universal patterns (OS files, editor files, AI tools, environment files, task management). Then add language-specific patterns for `{language/stack}`. Add English comments for each section."

### 7.2 `.gitattributes`

```gitattributes
# Enforce consistent line endings
* text=auto eol=lf

# Language-specific diff drivers
*.md text eol=lf diff=markdown linguist-documentation

# Binary files
*.png binary
*.jpg binary
*.pdf binary
```

> **Prompt to Claude Code:**
>
> "Create a `.gitattributes` file enforcing LF line endings. Add diff drivers for `{language/stack}` file types. Mark image and binary file extensions as binary."

---

## 8. Optional Extras

### 8.1 Changelog

> **Prompt to Claude Code:**
>
> "Create a `CHANGELOG.md` following the [Keep a Changelog format](https://keepachangelog.com/en/1.0.0/). Include an `[Unreleased]` section and one initial `[0.1.0]` entry with an `Added` subsection listing the initial project scaffolding."

### 8.2 GitHub Actions CI

> **Prompt to Claude Code:**
>
> "Create `.github/workflows/ci.yml` — a GitHub Actions workflow that:
> - Triggers on push to `main` and on pull requests
> - Runs on `ubuntu-latest`
> - Steps: checkout, setup `{language/runtime}`, install dependencies, run linter, run tests
> - Use matrix strategy for `{versions}` if multiple versions need support"

### 8.3 Funding Configuration

> **Prompt to Claude Code:**
>
> "Create `.github/FUNDING.yml` with entries for `{funding-platforms}` (e.g., github: username, ko_fi: username, custom: URL)."

### 8.4 Release Automation

> **Prompt to Claude Code:**
>
> "Create `.github/workflows/release.yml` — a GitHub Actions workflow that:
> - Triggers on push of version tags (`v*`)
> - Creates a GitHub Release with auto-generated release notes
> - Optionally publishes to `{package-registry}` (e.g., npm, PyPI, Maven Central)"

---

## 9. Dual AI Reviewer Setup

> A step-by-step technical guide to configuring Claude and Gemini as complementary PR reviewers on GitHub, with CODEOWNERS-based human approval as a mandatory merge gate.

### 9.1 Overview

#### The Problem

Open source repositories need consistent, thorough code review. AI-powered reviewers can provide instant feedback on every PR, but relying on a single bot creates a single point of failure and a narrow review perspective. More critically, AI reviews alone should never be sufficient to merge code into the main branch.

#### The Solution

This section establishes a **dual AI reviewer architecture** where:

- **Gemini Code Assist** provides inline code quality feedback and suggestions
- **Claude Code Review** performs structural review and submits formal APPROVE or REQUEST_CHANGES verdicts
- **CODEOWNERS** enforces mandatory human approval as the final merge gate

```text
PR Created
  |
  +-- Gemini Code Assist --> Inline comments, suggestions
  |
  +-- Claude Code Review --> APPROVE / REQUEST_CHANGES
  |
  +-- Code Owner (human) --> Final APPROVE required
  |
  v
Merge Allowed
```

#### Prerequisites

- GitHub repository with admin access
- GitHub Actions enabled
- [Gemini Code Assist](https://github.com/apps/gemini-code-assist) GitHub App installed
- [Claude Code](https://github.com/apps/claude-code) GitHub App installed (or OAuth token configured)

### 9.2 Architecture

#### Role Separation

| Reviewer | Type | Permission | Role |
|----------|------|------------|------|
| Gemini Code Assist | GitHub App (reviewer) | Automatic | Inline feedback, code suggestions, summary |
| Claude Code Review | GitHub Actions workflow | `pull-requests: write` | Formal review verdict (APPROVE / REQUEST_CHANGES) |
| Claude Code (mention) | GitHub Actions workflow | `pull-requests: write` | On-demand assistance via `@claude` |
| Code Owner (human) | CODEOWNERS | Required approval | Final merge authorization |

#### Why Two AI Reviewers?

Each reviewer brings different strengths:

- **Gemini** excels at granular code suggestions, style consistency checks, and inline comment-level feedback
- **Claude** excels at holistic review, architectural assessment, and formal approval decisions
- **Together** they cover both micro (line-level) and macro (design-level) review dimensions

#### Safety Model

```text
AI Reviewers (Claude + Gemini)
  = Advisory + Formal Check

Code Owner (human)
  = Mandatory Gate (cannot be bypassed by bots)
```

Even if both AI reviewers approve, the PR cannot merge without the code owner's explicit approval. This prevents automated merges of potentially harmful contributions.

### 9.3 Step-by-Step Setup

#### 9.3.1 Install Gemini Code Assist

1. Navigate to [github.com/apps/gemini-code-assist](https://github.com/apps/gemini-code-assist)
2. Click **Install** and select your repository
3. Gemini automatically activates as a PR reviewer upon installation

Gemini requires no workflow file. It operates as a GitHub App with native reviewer capabilities and will automatically:

- Post a summary comment on new PRs
- Leave inline code review comments
- Respond to `/gemini` commands in PR comments

#### 9.3.2 Configure Claude Code Action

Claude operates through GitHub Actions. Two workflows are needed:

##### Workflow 1: Automatic PR Review

Create `.github/workflows/claude-code-review.yml`:

```yaml
name: Claude Code Review

on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]

jobs:
  claude-review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write  # CRITICAL: enables APPROVE/REQUEST_CHANGES
      issues: read
      id-token: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - name: Run Claude Code Review
        id: claude-review
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          plugin_marketplaces: 'https://github.com/anthropics/claude-code.git'
          plugins: 'code-review@claude-code-plugins'
          prompt: >-
            /code-review:code-review
            ${{ github.repository }}/pull/${{ github.event.pull_request.number }}
```

**Key configuration: `pull-requests: write`**

The default template from Claude's GitHub integration sets `pull-requests: read`. This allows Claude to read PR content and post check results, but it **cannot** submit review verdicts (APPROVE or REQUEST_CHANGES). Changing this to `write` is the single most important configuration in this guide.

| Permission | Claude Can Do |
|------------|---------------|
| `pull-requests: read` | Read PR diff, post check pass/fail |
| `pull-requests: write` | All of the above + submit APPROVE / REQUEST_CHANGES reviews |

##### Workflow 2: On-Demand Assistance (`@claude`)

Create `.github/workflows/claude.yml`:

```yaml
name: Claude Code

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
  pull_request_review:
    types: [submitted]

jobs:
  claude:
    if: |
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review' && contains(github.event.review.body, '@claude')) ||
      (github.event_name == 'issues' && (contains(github.event.issue.body, '@claude') || contains(github.event.issue.title, '@claude')))
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write   # Enables review actions
      issues: write          # Enables issue comment responses
      id-token: write
      actions: read
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - name: Run Claude Code
        id: claude
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          additional_permissions: |
            actions: read
```

##### Store the OAuth Token

1. Go to **Settings > Secrets and variables > Actions** in your repository
2. Click **New repository secret**
3. Name: `CLAUDE_CODE_OAUTH_TOKEN`
4. Value: your Claude Code OAuth token (obtain from [Claude Code dashboard](https://console.anthropic.com/))

#### 9.3.3 Add CODEOWNERS

Create `.github/CODEOWNERS`:

```text
# Default code owner for all files
# Owner approval is always required regardless of bot reviews
* @your-github-username
```

Replace `@your-github-username` with the repository owner or maintainer team.

For repositories with multiple maintainers or domain-specific ownership:

```text
# Default owner
* @org/core-team

# Domain-specific owners
docs/           @org/docs-team
src/api/        @org/backend-team
src/components/ @org/frontend-team
.github/        @org/devops-team
```

#### 9.3.4 Configure Repository Ruleset

Go to **Settings > Rules > Rulesets** and create a new ruleset:

| Setting | Value | Reason |
|---------|-------|--------|
| Name | Main Branch Protection | Descriptive identifier |
| Enforcement | Active | Rules are enforced |
| Target | Default branch | Protects `main` |
| Restrict deletions | Enabled | Prevent branch deletion |
| Block force pushes | Enabled | Prevent history rewrite |
| Require pull request | Enabled | All changes via PR |
| Required approvals | 1 (minimum) | At least one approval needed |
| Require code owner review | **Enabled** | Human approval mandatory |
| Dismiss stale reviews | Optional | Re-review after new pushes |
| Require review thread resolution | Optional | Force addressing comments |

**Critical setting: `Require code owner review = true`**

This ensures that even if Claude submits an APPROVE, the PR remains blocked until the code owner (human) also approves. This is the safety mechanism that prevents fully automated merges.

##### Bypass Actors

Configure bypass actors for legitimate administrative needs:

- **Repository Admin** — can merge in emergencies (initial setup, CI fixes)
- **Maintain role** — same as admin for maintenance tasks

Bypass should be used sparingly and only for:

- Initial repository setup (bootstrapping PRs before reviewers are configured)
- CI/CD pipeline fixes that are blocked by circular dependencies
- Emergency hotfixes with post-merge review

### 9.4 How It Works in Practice

#### Normal PR Flow

```text
1. Contributor opens PR
   |
2. Gemini Code Assist (automatic)
   +-- Posts summary comment
   +-- Leaves inline code suggestions
   +-- State: COMMENTED (no formal verdict)
   |
3. Claude Code Review (automatic)
   +-- Analyzes full PR diff
   +-- Posts review with APPROVE or REQUEST_CHANGES
   +-- State: APPROVED or CHANGES_REQUESTED
   |
4. Contributor addresses feedback (if any)
   +-- Pushes fixes
   +-- Claude re-reviews on synchronize event
   |
5. Code Owner reviews
   +-- Reads AI feedback as input
   +-- Performs final human judgment
   +-- Submits APPROVE
   |
6. PR merges
```

#### Addressing AI Review Feedback

When Claude or Gemini leaves feedback:

1. **Fix the code** — push new commits to the branch
2. **Reply to the comment** — explain what was fixed and reference the commit
3. **Mark as resolved** — if review thread resolution is required

Example reply format:

```text
Fixed in abc1234 — updated placeholder to `[e.g., ...]` format.
```

#### On-Demand Claude Assistance

In any PR comment or issue, mention `@claude` to get help:

```text
@claude Can you review the error handling in this function?
@claude Suggest a more efficient algorithm for this sorting logic.
@claude What security concerns exist in this authentication flow?
```

### 9.5 Troubleshooting

#### Claude Review Shows as Check Pass/Fail Instead of Review

**Cause**: `pull-requests: read` permission in the workflow file.

**Fix**: Change to `pull-requests: write` in `.github/workflows/claude-code-review.yml`.

Note that `pull_request` event workflows use the workflow definition from the **merge base (main branch)**, not from the PR branch. This means permission changes only take effect after the PR containing those changes is merged into main.

#### PR is BLOCKED Despite AI Approval

**Cause**: `require_code_owner_review: true` in the ruleset. This is expected behavior.

**Fix**: The code owner (human) must approve. This is the intended safety mechanism.

#### Gemini Does Not Submit APPROVE

**Expected behavior**: Gemini Code Assist operates as a commenting reviewer. It provides inline suggestions and summaries but does not submit formal APPROVE or REQUEST_CHANGES verdicts. Claude fills this role.

#### Workflow Not Triggering on New Push

**Cause**: GitHub Actions `pull_request` event reads workflow definitions from the merge base branch, not the PR branch.

**Fix**: Workflow file changes must be merged to main before they take effect on subsequent PRs. For the initial setup PR, use admin bypass to merge.

#### `CLAUDE_CODE_OAUTH_TOKEN` Not Working

**Checklist**:

1. Verify the secret is set in **Settings > Secrets and variables > Actions**
2. Confirm the secret name matches exactly: `CLAUDE_CODE_OAUTH_TOKEN`
3. Check that the token has not expired
4. Verify the Claude Code GitHub App is installed on the repository

### 9.6 Security Considerations

#### Token Management

- Store `CLAUDE_CODE_OAUTH_TOKEN` as a repository secret, never in code
- Rotate tokens periodically according to your organization's security policy
- Use environment-scoped secrets for organizations with multiple environments

#### Permission Minimization

The workflows use the minimum permissions required:

| Permission | Scope | Justification |
|------------|-------|---------------|
| `contents: read` | Both workflows | Read repository code for review |
| `pull-requests: write` | Both workflows | Submit review verdicts |
| `issues: write` | `claude.yml` only | Respond to `@claude` mentions in issues |
| `id-token: write` | Both workflows | OIDC authentication for Claude |
| `actions: read` | `claude.yml` only | Read CI results for context |

#### Fork PR Considerations

For open source repositories accepting contributions from forks:

- GitHub Actions workflows triggered by `pull_request` from forks run with **read-only** permissions by default
- The `CLAUDE_CODE_OAUTH_TOKEN` secret is **not available** to fork PRs
- Consider using `pull_request_target` with caution if you need AI reviews on fork PRs
- Always require code owner approval as the final gate regardless of CI status

### 9.7 Customization Options

#### Limiting Claude Review Scope

To review only specific file types:

```yaml
on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]
    paths:
      - "src/**/*.ts"
      - "src/**/*.tsx"
      - "*.md"
```

#### Filtering by Contributor Type

To run Claude review only for external contributors:

```yaml
jobs:
  claude-review:
    if: |
      github.event.pull_request.author_association == 'FIRST_TIME_CONTRIBUTOR' ||
      github.event.pull_request.author_association == 'NONE'
```

#### Adding Gemini Customization

Create `.gemini/` directory in the repository root with configuration files:

```text
.gemini/
  config.yaml       # Review behavior settings
  style-guide.md    # Custom code review style guide
```

See [Gemini Code Assist documentation](https://developers.google.com/gemini-code-assist/docs/customize-gemini-behavior-github) for configuration options.

### 9.8 Quick Start Checklist

Use this checklist when setting up a new repository:

```markdown
[ ] Install Gemini Code Assist GitHub App
[ ] Install Claude Code GitHub App (or configure OAuth token)
[ ] Store CLAUDE_CODE_OAUTH_TOKEN as repository secret
[ ] Create .github/workflows/claude-code-review.yml (pull-requests: write)
[ ] Create .github/workflows/claude.yml (pull-requests: write, issues: write)
[ ] Create .github/CODEOWNERS with repository owner(s)
[ ] Create repository ruleset:
    [ ] Require pull request before merging
    [ ] Required approvals: 1+
    [ ] Require code owner review: enabled
    [ ] Restrict deletions: enabled
    [ ] Block force pushes: enabled
[ ] Verify setup with a test PR:
    [ ] Gemini posts summary and inline comments
    [ ] Claude submits APPROVE or REQUEST_CHANGES
    [ ] PR remains BLOCKED until code owner approves
    [ ] PR merges after code owner approval
```

### 9.9 Reference

#### File Structure

```text
.github/
  CODEOWNERS                              # Human approval gate
  workflows/
    claude-code-review.yml                # Auto review on PR events
    claude.yml                            # @claude mention handler
```

#### Related Documentation

- [Claude Code Action](https://github.com/anthropics/claude-code-action) — GitHub Actions integration
- [Claude Code Plugins](https://github.com/anthropics/claude-code/tree/main/plugins) — Review plugins
- [Gemini Code Assist](https://developers.google.com/gemini-code-assist/docs/review-github-code) — Setup and customization
- [GitHub Rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets) — Branch protection configuration
- [CODEOWNERS Syntax](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners) — Ownership patterns

---

## 10. Full Bootstrap Prompt

Use this single comprehensive prompt to generate all essential files in one session:

> **Master Prompt to Claude Code:**
>
> "Bootstrap a new open source project with the following specifications:
>
> - **Project name**: `{project-name}`
> - **Description**: `{one-line-description}`
> - **Primary language/stack**: `{language/stack}`
> - **License**: `{license-type}`
> - **Copyright holder**: `{name-or-organization}`
> - **Copyright year**: `{year}`
> - **Security contact**: `{security-contact-email}`
> - **Default commit type**: `{type}` (e.g., `feat` for software, `docs` for documentation)
>
> Generate the following files in order:
>
> 1. `LICENSE` — full license text
> 2. `.gitignore` — OS, editor, AI tools, env files, and `{language/stack}`-specific patterns
> 3. `.gitattributes` — LF enforcement, language-specific diff drivers, binary file declarations
> 4. `CODE_OF_CONDUCT.md` — Contributor Covenant v2.1
> 5. `CONTRIBUTING.md` — fork workflow, commit conventions, PR guidelines
> 6. `SECURITY.md` — vulnerability reporting, disclosure policy
> 7. `README.md` — full structure with project description, quick start placeholder, contributing link, license
> 8. `CLAUDE.md` — project rules, repository structure, git rules (including a '### Prohibited Actions' section), workflow guidelines, task management, core principles
> 9. `.github/ISSUE_TEMPLATE/bug-report.yml` — YAML form template
> 10. `.github/ISSUE_TEMPLATE/feature-request.yml` — YAML form template
> 11. `.github/PULL_REQUEST_TEMPLATE.md` — summary, related issue, changes, checklist
>
> **Optional — Dual AI Reviewer Setup (see Section 9):**
>
> 12. `.github/workflows/claude-code-review.yml` — automatic PR review workflow (`pull-requests: write`)
> 13. `.github/workflows/claude.yml` — on-demand `@claude` mention handler (`pull-requests: write`, `issues: write`)
> 14. `.github/CODEOWNERS` — human approval gate with repository owner(s)
>
> After generating all files, run `git add` for each file individually and commit with the message:
> `{type}(repo): initialize open source project structure`
>
> Do NOT use `git add -A` or `git add .`."

---

## 11. Post-Bootstrap Checklist

After Claude Code generates the files, verify completeness:

### File Existence

- [ ] `LICENSE` exists and contains the correct license text
- [ ] `.gitignore` covers OS, editor, AI tools, env files, and language-specific patterns
- [ ] `.gitattributes` enforces LF and sets diff drivers
- [ ] `CODE_OF_CONDUCT.md` includes reporting contact
- [ ] `CONTRIBUTING.md` includes fork workflow and commit conventions
- [ ] `SECURITY.md` includes vulnerability reporting instructions
- [ ] `README.md` has all major sections
- [ ] `CLAUDE.md` has project rules and git rules
- [ ] `.github/ISSUE_TEMPLATE/bug-report.yml` is valid YAML
- [ ] `.github/ISSUE_TEMPLATE/feature-request.yml` is valid YAML
- [ ] `.github/PULL_REQUEST_TEMPLATE.md` has checklist

### Dual AI Reviewer Files (Optional — Section 9)

- [ ] `.github/workflows/claude-code-review.yml` exists with `pull-requests: write` permission
- [ ] `.github/workflows/claude.yml` exists with `pull-requests: write` and `issues: write` permissions
- [ ] `.github/CODEOWNERS` exists with repository owner(s)
- [ ] `CLAUDE_CODE_OAUTH_TOKEN` is stored as a repository secret
- [ ] Gemini Code Assist GitHub App is installed on the repository
- [ ] Claude Code GitHub App is installed (or OAuth token configured)

### Content Quality

- [ ] License year and copyright holder are correct
- [ ] All placeholder values (`{project-name}`, etc.) have been replaced
- [ ] Commit message format in `CONTRIBUTING.md` matches `CLAUDE.md`
- [ ] Git rules in `CLAUDE.md` prohibit `git add -A`, `git add .`, `git push --force`, and `git reset --hard`
- [ ] `.gitignore` includes `.env`, `.env.local`, `.env.*.local`, `.DS_Store`, editor patterns (`.vscode/`, `.idea/`, `*.code-workspace`), `.claude/`, and `tasks/`
- [ ] Branch naming conventions are documented
- [ ] Contact information is consistent across `CODE_OF_CONDUCT.md` and `SECURITY.md`

### GitHub Repository Settings (Manual)

After pushing to GitHub, configure these settings in the repository:

- [ ] **General**: Set description and topics
- [ ] **Branches**: Add branch protection rule for `main` (require PR, require at least 1 review, require status checks to pass before merging, dismiss stale pull request approvals when new commits are pushed)
- [ ] **Pull Requests**: Enable "Allow squash merging" as default
- [ ] **Security**: Enable Dependabot alerts and security advisories
- [ ] **Pages** (optional): Enable GitHub Pages for documentation

### Dual AI Reviewer Repository Settings (Optional — Section 9)

- [ ] **Rules > Rulesets**: Create ruleset with require pull request, required approvals (1+), require code owner review enabled, restrict deletions, block force pushes
- [ ] **Bypass Actors**: Configure repository admin and maintain role for emergency merges
- [ ] Verify with a test PR: Gemini posts summary/inline comments, Claude submits APPROVE or REQUEST_CHANGES, PR remains BLOCKED until code owner approves, PR merges after code owner approval

---

## 12. Verification

Run these commands to verify the bootstrapped repository:

```bash
# Verify essential root files exist
for f in LICENSE README.md CLAUDE.md .gitignore .gitattributes; do
  test -f "$f" || { echo "❌ Missing essential file: $f" >&2; exit 1; }
done

# Verify community health files exist (in root, .github/, or docs/)
for f in CODE_OF_CONDUCT.md CONTRIBUTING.md SECURITY.md; do
  test -f "$f" || test -f ".github/$f" || test -f "docs/$f" || { echo "❌ Missing community health file: $f" >&2; exit 1; }
done

# Verify GitHub templates exist
for f in .github/PULL_REQUEST_TEMPLATE.md .github/ISSUE_TEMPLATE/bug-report.yml .github/ISSUE_TEMPLATE/feature-request.yml; do test -f "$f" || { echo "❌ Missing GitHub template: $f" >&2; exit 1; }; done

# Verify .gitignore covers essentials
for pattern in '\.env' '\.env\.local' '\.env\..*\.local' '\.DS_Store' '\.vscode/' '\.idea/' '\*\.code-workspace' '\.claude/' 'tasks/'; do
  grep -qE "^\s*${pattern}" .gitignore || { echo "❌ .gitignore is missing essential pattern: ${pattern//\\/}" >&2; exit 1; }
done

# Verify .gitattributes enforces LF
grep -qE '^\s*\*.*eol=lf' .gitattributes || { echo "❌ .gitattributes is not enforcing LF line endings" >&2; exit 1; }

# Verify git rules in CLAUDE.md
for cmd in "git add -A" "git add ." "git push --force" "git reset --hard"; do
  awk '/^##+/{p=0} /^### Prohibited Actions/{p=1;next} p' CLAUDE.md | sed 's/`//g' | grep -qF "$cmd" || { echo "❌ CLAUDE.md does not list the prohibited command '$cmd' under the '### Prohibited Actions' section" >&2; exit 1; }
done

# Verify dual AI reviewer files (optional — Section 9)
for f in .github/workflows/claude-code-review.yml .github/workflows/claude.yml .github/CODEOWNERS; do
  if test -f "$f"; then
    echo "✅ Dual AI reviewer file exists: $f"
  else
    echo "ℹ️  Optional dual AI reviewer file not found: $f (see Section 9 to set up)"
  fi
done

# Verify claude-code-review.yml has pull-requests: write (if file exists)
if test -f .github/workflows/claude-code-review.yml; then
  grep -qE 'pull-requests:\s*write' .github/workflows/claude-code-review.yml || { echo "⚠️  .github/workflows/claude-code-review.yml should have 'pull-requests: write' permission" >&2; }
fi

# Verify claude.yml has pull-requests: write and issues: write (if file exists)
if test -f .github/workflows/claude.yml; then
  grep -qE 'pull-requests:\s*write' .github/workflows/claude.yml || { echo "⚠️  .github/workflows/claude.yml should have 'pull-requests: write' permission" >&2; }
  grep -qE 'issues:\s*write' .github/workflows/claude.yml || { echo "⚠️  .github/workflows/claude.yml should have 'issues: write' permission" >&2; }
fi

# Verify clean git status
git status

# Verify initial commit exists
git log --oneline -1
```

- [ ] All essential files exist
- [ ] `.gitignore` covers environment files and AI tool directories
- [ ] `.gitattributes` enforces LF line endings
- [ ] `CLAUDE.md` contains git safety rules
- [ ] Initial commit is clean with no untracked files
- [ ] (Optional) Dual AI reviewer workflow files have correct permissions
- [ ] (Optional) CODEOWNERS file specifies repository owner(s)
