# Tech Stack Version Reference

> The official core version standards for technology stacks used in Claude Ready Standards projects.
> This document acts as the **Single Source of Truth** and overrides the version tables in individual project documents.

**Last Updated: 2026-03-05**

---

## Version Table

**Integrated Technology Stack Master List (Feb 2026)**

This table represents the latest stable releases and key features based on official documentation as of February 28, 2026.

| **Category**           | **Technology**          | **Latest Version**             | **Key Features & Official URL**                                   |
| ---------------------- | ----------------------- | ------------------------------ | ----------------------------------------------------------------- |
| **Monorepo**           | **pnpm + Turborepo**    | pnpm `10.30.0`, turbo `2.3.3+` | Package mgmt + build caching — pnpm.io, turbo.build               |
| **Runtime**            | **Node.js**             | `v24.14.0 (LTS)`               | Recommended stable LTS (Krypton) — nodejs.org                     |
| **Language**           | **TypeScript**          | `v5.7+`                        | Strict type checking enabled by default — typescriptlang.org      |
| **Language**           | **Kotlin**              | `v2.3.10`                      | K2 compiler stabilized, Java 25 support — kotlinlang.org          |
| **Language**           | **Python**              | `v3.14.3`                      | JIT compiler & performance improvements — python.org              |
| **Backend (JS)**       | **NestJS**              | `v11.1.14`                     | Security & stability patches for v11 — docs.nestjs.com            |
| **Backend (Python)**   | **FastAPI**             | `v0.133.1`                     | Max async performance with Pydantic v2 — fastapi.tiangolo.com     |
| **Backend (Kotlin)**   | **Spring Boot**         | `v4.0.3`                       | Spring Framework 7, Jakarta EE 11, Kotlin first-class — spring.io |
| **Web Frontend**       | **Next.js + React**     | Next `v16.1.6`, React `v19`    | AI feature integration & RSC optimization — nextjs.org            |
| **Styling**            | **Tailwind CSS**        | `v4.2.1`                       | High-speed builds with Rust engine — tailwindcss.com              |
| **Client**             | **Flutter**             | SDK `3.41+`                    | Cross-platform UI toolkit for mobile, web & desktop — flutter.dev |
| **ORM (SQL)**          | **Prisma ORM**          | `v7.4.1`                       | Query optimization & engine enhancements — prisma.io              |
| **ORM (Kotlin)**       | **Exposed**             | `v1.0.0`                       | **First Major Release**, R2DBC support \* — JetBrains Exposed     |
| **ORM (Python SQL)**   | **SQLAlchemy**          | `v2.0 async`                   | Async-first ORM with Pydantic v2 integration — sqlalchemy.org     |
| **ODM (NoSQL)**        | **Mongoose**            | `v8.x`                         | MongoDB ODM with enhanced TS support — mongoosejs.com             |
| **ODM (Python NoSQL)** | **Motor**               | `v3.7.x`                       | Async MongoDB driver for Python — motor.readthedocs.io            |
| **State Mgmt**         | **Tanstack Query**      | `v5.66.0+`                     | Industry standard for server state mgmt — tanstack.com            |
| **Client State**       | **Zustand**             | `v5.0.x`                       | Lightweight client state for consumer web — zustand.docs.pmnd.rs  |
| **Forms**              | **React Hook Form**     | `v7.71.x`                      | Performant form state & validation — react-hook-form.com          |
| **Animation**          | **Framer Motion**       | `v12.x`                        | Production React animation library — motion.dev                   |
| **Charts**             | **Recharts**            | `v3.7.x`                       | Composable React charting — recharts.org                          |
| **Icons**              | **Lucide React**        | `v0.577.x`                     | Lightweight SVG icon set — lucide.dev                             |
| **PWA**                | **Serwist**             | `v9.5.x`                       | Next.js Service Worker toolkit — serwist.pages.dev                |
| **Push**               | **Firebase (FCM)**      | `v12.x`                        | Web Push via Firebase Cloud Messaging — firebase.google.com       |
| **Message Queue**      | **BullMQ + Redis**      | BullMQ `v5`, Redis `v7.4.x`    | High-performance Redis-based task queues — bullmq.io              |
| **Task Queue (Py)**    | **arq**                 | `v0.26.x`                      | Async Redis-based task queue for Python — arq-docs.helpmanual.io  |
| **DB (Relational)**    | **PostgreSQL**          | `v17.2`                        | Parallel query processing & security — postgresql.org             |
| **DB (Document)**      | **MongoDB**             | `v8.0.x`                       | Advanced Vector Search & data optimization — mongodb.com          |
| **Cache/Store**        | **Redis**               | `v7.4.x`                       | Improved cluster stability & persistence — redis.io               |
| **API Docs**           | **Swagger UI**          | `v5.x`                         | Full OpenAPI 3.1 specification support — swagger.io               |
| **IaC**                | **Terraform**           | `v1.11.x`                      | Standard for Infrastructure as Code — terraform.io                |
| **Container**          | **Docker + Compose**    | `v28.0.x`                      | Build optimization & environment isolation — docs.docker.com      |
| **OS**                 | **Ubuntu**              | `24.04 LTS`                    | Noble Numbat, Long Term Support — ubuntu.com                      |
| **Testing**            | **Vitest / Playwright** | Latest Stable                  | Next-gen testing tools — vitest.dev, playwright.dev               |
| **AI Agentic Base**    | **Claude Code**         | `Opus 4.6`                     | High-performance AI coding agent — anthropic.com                  |

**Quick Usage Guide**

- **Exposed 1.0.0 Warning**: Contains **breaking changes** regarding package names and structure. Check the Migration Guide before upgrading.
- **Tailwind CSS v4**: Shifted to a **CSS-first configuration**. Review the official v4 Upgrade Guide for new project setups.

---

## Relationship with `boilerplate-guide/00-index.md`

A tech stack summary table also exists in `boilerplate-guide/00-index.md`.
The roles of the two documents are categorized as follows:

| Document                          | Role                                                                                                  |
| --------------------------------- | ----------------------------------------------------------------------------------------------------- |
| **TECH-STACK.md** (This document) | Baseline for the latest versions — update this document first when versions change                    |
| `boilerplate-guide/00-index.md`   | Tech stack summary in the context of the Boilerplate Guide — synchronize after updating TECH-STACK.md |

If a version discrepancy is found, **TECH-STACK.md is considered the definitive source**.

---

## Version Update Procedure

Review the versions according to the procedure below at least once a quarter:

1. **Research** — Verify the latest stable versions for each technology.
2. **Update TECH-STACK.md** — Update the version table and the `Last Updated` date in this document.
3. **Synchronize Child Documents** — Reflect the changes in referenced documents, such as `boilerplate-guide/00-index.md`.
4. **Commit** — Commit using the format `docs(tech-stack): update versions for YYYY-QN`.
