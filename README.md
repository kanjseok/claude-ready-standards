# Claude Ready Standards

> Pre-built technical standards optimized for Claude Code.
> Skip the tech stack decisions — start building immediately after your PRD is ready.

---

## Why Claude Ready Standards?

When your PRD and technical design are complete, the next step — choosing specific technologies, defining base architecture, configuring toolchains — still consumes significant time and mental energy.

**Claude Ready Standards eliminates that decision fatigue.** This repository provides a curated set of technical standards, boilerplate configurations, and architectural patterns that are ready to use with Claude Code out of the box.

- **Problem**: Even with a solid PRD, teams spend days deciding on tech specs, folder structures, and baseline configurations.
- **Solution**: A comprehensive, copy-and-paste-ready standard framework that lets you go from design to implementation with Claude Code — immediately.
- **Scope**: Monorepo configuration, backend/frontend/client setups, testing strategies, security, CI/CD, and Git workflows.

---

## Built Exclusively for Claude Code

> **Warning**: All automation workflows in this repository are designed **exclusively for Claude Code**.

The rules, agent configurations, hook settings, and CLAUDE.md patterns defined within Claude Ready Standards are fully optimized for the functionalities of Claude Code.

If you use other AI coding tools:

- The hierarchical structure of CLAUDE.md will be ignored.
- Rules, Hooks, and Agents configurations will not work.
- Automatic application of commit message formats, code reviews, and TDD workflows is not guaranteed.
- **The consistency and structure of the standard may be compromised.**

---

## Standards Catalog

### Active Standards

| Category          | Directory                                    | Description                                                                                                                                   |
| ----------------- | -------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| Boilerplate Guide | [`boilerplate-guide/`](./boilerplate-guide/) | Comprehensive bootstrapping guide for full-stack projects using pnpm monorepo + NestJS + FastAPI + Kotlin + Next.js + Flutter (19 documents). |

### Potential Future Categories

| Category             | Description                                            |
| -------------------- | ------------------------------------------------------ |
| `api-design/`        | REST/GraphQL API design standards                      |
| `deployment/`        | Deployment strategies and infrastructure configuration |
| `code-style/`        | Language-specific code style guides                    |
| `incident-response/` | Incident response procedures                           |

---

## Key References

| Document                                                           | Description                                               |
| ------------------------------------------------------------------ | --------------------------------------------------------- |
| [`TECH-STACK.md`](./TECH-STACK.md)                                 | Baseline for tech stack versions (Single Source of Truth) |
| [`CLAUDE.md`](./CLAUDE.md)                                         | Rules for working with Claude Code within this repository |
| [`OPEN-SOURCE-STARTER-GUIDE.md`](./OPEN-SOURCE-STARTER-GUIDE.md) | Guide to bootstrap new OSS projects with Claude Code |
| [`boilerplate-guide/00-index.md`](./boilerplate-guide/00-index.md) | Index for the Boilerplate Guide                           |

---

## Document Writing Principles

All documents in this repository adhere to the following principles:

1. **Exclusion of Business Logic** — Covers only technical infrastructure and development rules; not tied to any specific business domain.
2. **Copy-and-Paste Ready** — Includes actual configuration files and commands as code blocks for immediate application.
3. **Self-Contained** — Includes necessary context internally so each document can be referenced independently.
4. **Includes Verification** — Provides verification commands or checklists at the end of each document to confirm successful setups.
5. **English** — All documents, configurations, code snippet comments, and variable names must be written in English.

---

## Contributing

We welcome contributions! Please read our [Contributing Guide](CONTRIBUTING.md) before submitting a PR.

This project follows the [Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md).

---

## License

This work is licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE).

You are free to share and adapt the material for any purpose, even commercially, as long as you give appropriate credit.

---

## Author

Created and maintained by [kanjseok](https://github.com/kanjseok).
