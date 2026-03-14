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

### 3.1 Code of Conduct

> **Prompt to Claude Code:**
>
> "Create a `CODE_OF_CONDUCT.md` based on the Contributor Covenant v2.1. Include sections for: Our Pledge, Our Standards (positive and negative examples), Enforcement Responsibilities, Scope, Reporting (via GitHub Issues and maintainer contact), and Attribution. Use `{contact-method}` as the reporting contact."

### 3.2 Contributing Guide

> **Prompt to Claude Code:**
>
> "Create a `CONTRIBUTING.md` with the following sections:
>
> 1. **Before You Start** — search existing issues, open issues for non-trivial changes, link to Code of Conduct
> 2. **How to Contribute** — fork & branch workflow with commands, branch naming conventions (`feature/`, `fix/`, `chore/`)
> 3. **Commit Message Convention** — Conventional Commits format with `{default-type}` as the default type, list of scopes, and examples of correct/incorrect messages
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
> - Fields: Affected area (input, required), Issue type (dropdown: incorrect behavior, crash, performance, security, other), Description (textarea, required), Steps to reproduce (textarea, required), Expected behavior (textarea), Environment info (textarea: OS, language version, project version)
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
> - Prohibited git commands (git add -A, git add ., force push, hard reset)
> - Commit message format with examples
> - Branch naming conventions
>
> **Section 4 — Work Workflow:**
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
> ```"

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
> "Create a `README.md` for `{project-name}` with the structure above. The project is `{one-line-description}`. Fill in the Why section with: Problem — `{problem}`, Solution — `{solution}`, Scope — `{scope}`. Add placeholder sections for Quick Start, Documentation, Contributing, and License. Use `{license-type}` for the license section."

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
> "Create a `CHANGELOG.md` following the Keep a Changelog format (keepachangelog.com). Include an `[Unreleased]` section and one initial `[0.1.0]` entry with an `Added` subsection listing the initial project scaffolding."

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

## 9. Full Bootstrap Prompt

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
> - **Security contact**: `{email}`
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
> 8. `CLAUDE.md` — project rules, repository structure, git rules, workflow guidelines, task management, core principles
> 9. `.github/ISSUE_TEMPLATE/bug-report.yml` — YAML form template
> 10. `.github/ISSUE_TEMPLATE/feature-request.yml` — YAML form template
> 11. `.github/PULL_REQUEST_TEMPLATE.md` — summary, related issue, changes, checklist
>
> After generating all files, run `git add` for each file individually and commit with the message:
> `{default-type}(repo): initialize open source project structure`
>
> Do NOT use `git add -A` or `git add .`."

---

## 10. Post-Bootstrap Checklist

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

### Content Quality

- [ ] License year and copyright holder are correct
- [ ] All placeholder values (`{project-name}`, etc.) have been replaced
- [ ] Commit message format in `CONTRIBUTING.md` matches `CLAUDE.md`
- [ ] Git rules in `CLAUDE.md` prohibit `git add -A`, `git add .`, `git push --force`, and `git reset --hard`
- [ ] `.gitignore` includes `.env` and `tasks/`
- [ ] Branch naming conventions are documented
- [ ] Contact information is consistent across `CODE_OF_CONDUCT.md` and `SECURITY.md`

### GitHub Repository Settings (Manual)

After pushing to GitHub, configure these settings in the repository:

- [ ] **General**: Set description and topics
- [ ] **Branches**: Add branch protection rule for `main` (require PR, require reviews)
- [ ] **Pull Requests**: Enable "Allow squash merging" as default
- [ ] **Security**: Enable Dependabot alerts and security advisories
- [ ] **Pages** (optional): Enable GitHub Pages for documentation

---

## Verification

Run these commands to verify the bootstrapped repository:

```bash
# Verify all essential files exist
ls -la LICENSE CODE_OF_CONDUCT.md CONTRIBUTING.md SECURITY.md README.md CLAUDE.md .gitignore .gitattributes

# Verify GitHub templates exist
ls -la .github/PULL_REQUEST_TEMPLATE.md .github/ISSUE_TEMPLATE/bug-report.yml .github/ISSUE_TEMPLATE/feature-request.yml

# Verify .gitignore covers essentials
grep -E "\.env|\.DS_Store|\.claude|tasks/" .gitignore

# Verify .gitattributes enforces LF
grep "eol=lf" .gitattributes

# Verify git rules in CLAUDE.md
grep -E "git add -A|git add \.|git push --force|git reset --hard" CLAUDE.md

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
