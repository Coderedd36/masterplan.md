# CODY MASTERPLAN — fixd-engine

> **Purpose:** This document is the single source of truth for how Cody (the AI coding assistant) should understand, extend, and maintain the `fixd-engine` project. It captures architecture decisions, coding conventions, feature roadmap, and operational guidance.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture](#2-architecture)
3. [Tech Stack](#3-tech-stack)
4. [Directory Structure](#4-directory-structure)
5. [Coding Conventions](#5-coding-conventions)
6. [Feature Roadmap](#6-feature-roadmap)
7. [Testing Strategy](#7-testing-strategy)
8. [CI/CD Pipeline](#8-cicd-pipeline)
9. [Security & Compliance](#9-security--compliance)
10. [Contribution Guidelines](#10-contribution-guidelines)
11. [Glossary](#11-glossary)

---

## 1. Project Overview

`fixd-engine` is a diagnostic and repair orchestration engine designed to automate the detection, triage, and resolution of software defects. It exposes a programmatic API that integrates with CI pipelines, issue trackers, and AI-assisted repair workflows.

**Core goals:**

- Automatically identify and classify bugs from logs, test output, and static analysis.
- Suggest or apply code fixes using AI-generated patches.
- Integrate with popular developer toolchains (GitHub Actions, Jira, Slack, VS Code).
- Provide full audit trails of every fix attempt.

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      fixd-engine                        │
│                                                         │
│  ┌──────────┐   ┌────────────┐   ┌───────────────────┐  │
│  │  Ingest  │──▶│  Analyzer  │──▶│  Repair Planner   │  │
│  └──────────┘   └────────────┘   └───────────────────┘  │
│        │               │                   │            │
│        ▼               ▼                   ▼            │
│  ┌──────────┐   ┌────────────┐   ┌───────────────────┐  │
│  │  Event   │   │  Defect    │   │  Patch Generator  │  │
│  │  Store   │   │  Registry  │   │  (AI-powered)     │  │
│  └──────────┘   └────────────┘   └───────────────────┘  │
│                                           │            │
│                                           ▼            │
│                               ┌───────────────────┐    │
│                               │   Validation &    │    │
│                               │   Apply Engine    │    │
│                               └───────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Responsibility |
|-----------|----------------|
| **Ingest** | Consumes events from CI webhooks, log streams, and static-analysis reports. |
| **Analyzer** | Classifies defects by type, severity, and affected module. |
| **Repair Planner** | Selects a repair strategy (auto-fix, human escalation, or suppression). |
| **Event Store** | Append-only log of all engine activity for auditability. |
| **Defect Registry** | Persistent store of known defects, states, and fix history. |
| **Patch Generator** | Calls LLM/AI service to produce candidate patches. |
| **Validation & Apply Engine** | Runs candidate patches through tests; applies passing patches. |

---

## 3. Tech Stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Runtime | Node.js 20 LTS | Wide ecosystem; async I/O suits event-driven core. |
| Language | TypeScript 5 | Type safety reduces defect introduction during development. |
| API layer | Fastify | High-performance HTTP; JSON schema validation built-in. |
| Queue | BullMQ + Redis | Reliable job processing with retry and backoff. |
| Database | PostgreSQL 16 | ACID transactions; JSONB for semi-structured defect metadata. |
| ORM | Prisma | Type-safe schema; migrations as code. |
| AI service | OpenAI GPT-4o (pluggable) | Patch generation; provider abstracted behind `AIProvider` interface. |
| Testing | Vitest | Fast; native ESM; compatible with TypeScript project setup. |
| Container | Docker + Compose | Reproducible local dev; mirrors production. |
| CI | GitHub Actions | Native integration with repository. |

---

## 4. Directory Structure

```
fixd-engine/
├── src/
│   ├── ingest/          # Webhook handlers & log parsers
│   ├── analyzer/        # Defect classification logic
│   ├── planner/         # Repair strategy selection
│   ├── patch/           # AI patch generation & diff utilities
│   ├── validation/      # Test runner integration & patch validation
│   ├── store/           # Event store & defect registry (DB layer)
│   ├── api/             # REST + GraphQL API routes
│   ├── config/          # Environment & feature-flag configuration
│   ├── shared/          # Types, utilities, constants
│   └── index.ts         # Application entry point
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── .github/
│   └── workflows/
├── docs/
│   └── adr/             # Architecture Decision Records
├── CODY_MASTERPLAN.md   # ← This file
├── CONTRIBUTING.md
├── package.json
└── tsconfig.json
```

---

## 5. Coding Conventions

### General

- All source files use **TypeScript**; `any` is banned (`noImplicitAny: true`).
- Prefer `async/await` over raw promise chains.
- No magic strings — use constants or enums defined in `src/shared/constants.ts`.
- Side-effectful operations (DB writes, HTTP calls, AI calls) must be behind interfaces so they can be mocked in tests.

### Naming

| Artifact | Convention | Example |
|----------|-----------|---------|
| Files | `kebab-case` | `defect-classifier.ts` |
| Classes | `PascalCase` | `DefectClassifier` |
| Functions / variables | `camelCase` | `classifyDefect()` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_RETRY_COUNT` |
| Database tables | `snake_case` | `defect_events` |

### Error Handling

- Every async function that calls external services must wrap calls in try/catch and emit a structured error to the event store.
- Errors are typed using a custom `FixdError` class that includes `code`, `message`, and optional `context`.
- Never swallow errors silently.

### Logging

- Use the shared `logger` instance (wraps `pino`).
- Log levels: `debug` (dev only) · `info` · `warn` · `error`.
- Include `traceId` and `defectId` in every log line when available.

### Commits

Follow **Conventional Commits**:

```
feat(patch): add retry logic for failed AI calls
fix(analyzer): correct severity mapping for ESLint rules
chore(deps): bump prisma to 5.x
docs(adr): record decision to use BullMQ
```

---

## 6. Feature Roadmap

### Phase 1 — Foundation (Current)

- [x] Repository scaffolding and CI setup
- [x] Core event store (PostgreSQL + Prisma)
- [x] Basic Ingest webhook for GitHub Actions
- [ ] Defect classifier (rule-based, severity levels 1–5)
- [ ] REST API with authentication (API key)
- [ ] Docker Compose local dev environment

### Phase 2 — AI Repair Loop

- [ ] `AIProvider` abstraction + OpenAI GPT-4o integration
- [ ] Patch generator: produce unified diffs from defect context
- [ ] Validation engine: apply patch to isolated container, run tests
- [ ] Auto-apply patches that pass validation (configurable threshold)
- [ ] Patch history UI (basic web dashboard)

### Phase 3 — Integrations

- [ ] GitHub PR auto-comment with fix suggestion
- [ ] Jira issue auto-update on fix
- [ ] Slack notification on fix applied / escalation
- [ ] VS Code extension for inline fix previews

### Phase 4 — Observability & Scale

- [ ] Metrics endpoint (Prometheus-compatible)
- [ ] Distributed tracing (OpenTelemetry)
- [ ] Multi-tenant support
- [ ] Horizontal scaling of validation workers

---

## 7. Testing Strategy

| Layer | Tool | Coverage Target |
|-------|------|----------------|
| Unit | Vitest | ≥ 90 % of `src/` |
| Integration | Vitest + test containers | All DB & queue interactions |
| E2E | Playwright (API) | All public API routes |

### Rules

- Every new function in `src/` must have a corresponding unit test.
- Integration tests run against a real PostgreSQL instance spun up via Docker.
- No test may call a real external AI service — use recorded fixtures or mocks.
- Tests are required before a PR is merged (CI enforced).

---

## 8. CI/CD Pipeline

```
push / PR
    │
    ├─► lint (ESLint + Prettier)
    ├─► type-check (tsc --noEmit)
    ├─► unit tests
    ├─► integration tests (docker-compose up)
    └─► build Docker image
            │
            └─► (main branch only)
                    ├─► push image to GHCR
                    └─► deploy to staging
```

All steps must pass before merge to `main`.

---

## 9. Security & Compliance

- **Secrets:** Never commit secrets. Use GitHub Actions secrets + environment variables.
- **Dependencies:** Run `npm audit` in CI; block merge on high/critical CVEs.
- **API authentication:** All endpoints require a valid API key passed as `Authorization: Bearer <key>`.
- **Input validation:** All API inputs validated with JSON schema (Fastify built-in).
- **AI output:** Patches generated by AI are never applied without passing automated validation tests.
- **Audit log:** Every fix attempt (success or failure) is written to the immutable event store.
- **GDPR:** No PII stored in defect metadata; logs are retained for 90 days then purged.

---

## 10. Contribution Guidelines

1. Fork the repository and create a feature branch: `git checkout -b feat/your-feature`.
2. Follow the coding conventions in §5.
3. Write tests before submitting a PR (TDD preferred).
4. Update `CODY_MASTERPLAN.md` if your change affects architecture or roadmap.
5. Open a PR against `main`; request a review from at least one maintainer.
6. Ensure all CI checks pass before requesting final review.

For larger changes, open an issue first to discuss the approach.

---

## 11. Glossary

| Term | Definition |
|------|-----------|
| **Defect** | A detected software bug or error, regardless of source. |
| **Patch** | A code change (unified diff) proposed to fix a defect. |
| **Fix attempt** | One lifecycle of: detect → plan → patch → validate → (apply or reject). |
| **Event store** | Append-only log of all engine actions; never mutated. |
| **Repair strategy** | The chosen action for a defect: auto-fix, escalate, or suppress. |
| **AI provider** | Pluggable interface to any LLM used for patch generation. |
| **Validation** | Running a candidate patch through automated tests in isolation. |

---

*Last updated: 2026-04-24 · Maintained by the fixd-engine core team*
