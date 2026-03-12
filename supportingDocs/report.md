# Mission Control — Complete Code Review Report
**Review Date:** 2026-03-12
**Repository:** https://github.com/AxiomOperator/mission-control
**Version Reviewed:** 2.0.0
**Reviewed By:** GitHub Copilot (Principal Engineer / Security Architect mode)
**Review Method:** Parallel multi-domain static analysis across 7 specialized review passes

---

## A. Executive Summary

### Overall Assessment

Mission Control is a **structurally sound, security-conscious, and well-documented** full-stack TypeScript application. For a dashboard in active development, it demonstrates engineering maturity well above the average open-source project at this stage — particularly in its security posture, API design, and container hardening. However, it has a fundamental split personality that must be understood:

> **It is an excellent metadata management and monitoring UI for agents and tasks. It is not yet a functional orchestration engine.**

The CRUD, auth, logging, and real-time infrastructure layers are close to production-ready. The *execution* layer — the part that actually dispatches work to agents, runs workflows, coordinates sub-agents, and manages scheduling — is largely absent or stubbed.

### Maturity Level

| Layer | Maturity | Notes |
|-------|----------|-------|
| API infrastructure | ★★★★☆ | Auth, RBAC, validation, logging — solid |
| Security posture | ★★★★☆ | CSP, injection guards, rate limiting — strong |
| Real-time / WebSocket | ★★★☆☆ | Architecture is sound; edge cases need fixing |
| Frontend UI | ★★★☆☆ | Feature-rich but monolithic and fragile |
| Testing | ★★★★☆ | 1,237 tests; needs coverage extension |
| Orchestration features | ★★☆☆☆ | Metadata yes; execution engine no |
| Type safety | ★★★☆☆ | 164+ suppressions undermine strict mode |
| DevEx / tooling | ★★★★★ | Exceptional installation, Docker, scripts |

### Biggest Strengths

1. Exceptional security architecture (CSP nonces, timing-safe comparisons, injection guards, RBAC)
2. Comprehensive REST API surface (131 routes, fully documented via OpenAPI)
3. Thoughtful real-time architecture (WebSocket + SSE + smart polling fallback)
4. Production-grade Docker setup with hardened compose overlay
5. 1,237 automated tests (Vitest + Playwright) with quality-gate CI
6. Excellent DevEx: one-command install, cross-platform, comprehensive docs

### Biggest Risks

1. **No task/workflow execution engine** — the core reason this product exists is not implemented
2. **Sub-agent support is 5% complete** — UI exists, backend is zero
3. **Monolithic UI components** — three components exceed 2,000 lines each; the main page is 567 lines
4. **WebSocket event sequence gaps cause silent data loss** — no recovery path exists
5. **164+ TypeScript suppressions** undermine the stated strict-mode policy
6. **SQLite single-writer model** is appropriate now but will be a scalability ceiling

### Verdict

> **"Continue cautiously with targeted cleanup"**

The project is structurally sound enough for continued feature development, but the execution layer gap is a fundamental architectural hole that will become increasingly expensive to retrofit as more UI is added on top of it. Address the orchestration execution engine before expanding the UI surface further.

---

## B. Repository Overview

### Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| Framework | Next.js (App Router) | 16.1.6 |
| Runtime | Node.js | 22.x (enforced) |
| Language | TypeScript | 5.7.2 (strict mode) |
| UI Library | React | 19.0.1 |
| State Management | Zustand | 5.0.11 |
| Styling | Tailwind CSS | 3.4.17 |
| Database | SQLite (better-sqlite3) | 12.6.2 |
| Logging | Pino | 10.3.1 |
| Validation | Zod | — |
| Testing (unit) | Vitest | — |
| Testing (e2e) | Playwright | — |
| Package Manager | pnpm | — |
| Container | Docker (multi-stage) | — |

### Frameworks / Key Libraries

- **Radix UI** — accessible primitive components
- **OpenClaw** — external AI agent gateway (primary integration target)
- **better-sqlite3** — synchronous SQLite driver
- **pino** — structured JSON logging
- **zxcvbn-style** enforcement candidates (not yet used)

### Major Directories

```
src/app/api/        131 API routes — auth, agents, tasks, workflows, chat, webhooks, admin
src/components/     88 React components — panels (42), UI primitives, layout, chat
src/lib/            18,100 lines — auth, db, adapters, websocket, injection-guard, rate-limit
src/store/          Zustand store (1,128 lines, 40+ properties)
src/types/          TypeScript interfaces
tests/              691 Playwright e2e tests (60 files)
src/lib/__tests__/  546 Vitest unit tests
skills/             2 built-in skills (mission-control-manage, installer)
docs/               5.4 MB operational documentation
ops/                Provisioner daemon scripts
scripts/            Build/deploy automation (1,493 lines)
```

### High-Level Architecture

Mission Control is a **Next.js monolith** with an embedded SQLite database, serving both the React frontend and the REST/SSE API from a single process. It connects outbound to OpenClaw gateway instances via WebSocket. The intended pattern is:

```
Browser ──(SSE/WebSocket)──► Next.js server ──(SQLite)──► Local state
                                             ──(WebSocket)──► OpenClaw Gateway ──► AI Agents
```

The gateway is external; Mission Control is the management plane. There is no message queue, no external cache, and no secondary services beyond optional Google OAuth.

---

## C. Strengths — What Is Done Well

### Security Architecture
The security posture is exceptional for a project at this stage. CSP with per-request nonces, timing-safe comparisons via `crypto.timingSafeEqual()`, 16-rule injection guard covering prompt injection / command injection / SSRF / SQL injection / exfiltration, parameterized database queries throughout, and a properly hardened Docker compose overlay. This is not boilerplate — it reflects deliberate security thinking.

### API Design
131 REST endpoints with consistent response envelopes, RBAC on every route via `requireRole()`, Zod schema validation on every mutation, structured Pino logging, and a full OpenAPI 3.1 specification. This is clean and professional.

### Real-Time Architecture
The three-layer fallback model (WebSocket → SSE → polling) with proper cleanup, reconnect backoff, and visibility-aware polling pause is architecturally thoughtful. Connection lifecycle management has no detected memory leaks across 17+ timer/listener pairs.

### Testing Investment
1,237 tests (546 unit + 691 e2e), centralized test helpers with factory functions, proper cleanup patterns, six dedicated security test files, and a full quality-gate CI pipeline (lint → typecheck → unit → build → e2e). This is well above average.

### Developer Experience
The `install.sh` (430 lines) is one of the best installation scripts reviewed — OS/arch detection, auto-selection of Docker vs local, secure `.env` generation, systemd service setup, OpenClaw health check. The Docker multi-stage build is correct. Documentation at 5.4 MB is comprehensive.

### Operational Hardening
Non-root Docker user (UID 1001), read-only root filesystem, `cap_drop: ALL`, resource limits, HSTS, secure cookie flags, WAL-mode SQLite — these demonstrate production intent, not just development shortcuts.

---

## D. Findings by Category

---

### D1. Architecture

| # | Title | Severity | File(s) | Issue | Impact |
|---|-------|----------|---------|-------|--------|
| A1 | No task/workflow execution engine | **Critical** | `src/app/api/tasks/`, `src/app/api/workflows/`, `src/lib/task-dispatch.ts` | Task dispatch functions exist but are never called. Workflow templates are stored but never instantiated. No execution lifecycle exists. | Core product value is absent. All orchestration is manual. |
| A2 | Sub-agent support is unimplemented | **Critical** | `src/types/index.ts`, `src/components/panels/agent-detail-tabs.tsx` | `type: 'subagent'` exists in the schema and UI. There is no backend endpoint, no lifecycle management, no task handoff. | Hierarchical orchestration is impossible. UI misleads users. |
| A3 | No repository/DAO layer | **High** | `src/app/api/*/route.ts` (131 files) | Database queries are written inline in every route handler. Schema changes require touching 131 files. | Any schema evolution is fragile and expensive. |
| A4 | No service layer | **High** | `src/app/api/*/route.ts` | Business logic is mixed with HTTP concerns in route handlers. No `lib/services/` layer exists. | Untestable business logic; routes become god-functions. |
| A5 | SQLite single-writer ceiling | **Medium** | `src/lib/db.ts` | Better-sqlite3 is synchronous, single-process. WAL helps concurrent reads but not concurrent writes. Not safe for PM2 cluster mode or multiple containers sharing one volume. | Hard scalability ceiling; must be understood by operators. |
| A6 | Framework adapters are stubs | **High** | `src/lib/adapters/crewai.ts`, `langgraph.ts`, `autogen.ts`, `claude-sdk.ts` | All non-OpenClaw adapters are copy-paste shells. They broadcast events but do not write to the database or implement framework-specific protocol. | Lock-in to OpenClaw; advertised multi-framework support is not real. |
| A7 | No API versioning strategy | **Medium** | `src/app/api/` | 131 routes with no `/v1/` prefix. Breaking changes will affect all clients. | Future API evolution will be disruptive. |
| A8 | No rollback for database migrations | **Medium** | `src/lib/migrations.ts` | Migrations are forward-only. A bad migration cannot be undone without manual intervention. | Deployment failures requiring rollback are unrecoverable automatically. |
| A9 | Agent name used as foreign key | **Medium** | `src/app/api/agents/route.ts`, task queries | Some queries join on `agent.name` rather than `agent.id`. Names can change; this creates referential integrity gaps. | Data inconsistency if agents are renamed. |
| A10 | Pipelines endpoints are empty stubs | **Critical** | `src/app/api/pipelines/route.ts`, `src/app/api/pipelines/run/route.ts` | Both endpoints are completely unimplemented. Listed in OpenAPI spec as supported. | Broken API contract. Any client expecting pipelines will receive nothing. |

---

### D2. Frontend / UI

| # | Title | Severity | File(s) | Issue | Impact |
|---|-------|----------|---------|-------|--------|
| F1 | Monolithic main page | **High** | `src/app/[[...panel]]/page.tsx` (567 lines) | 43 hardcoded panel imports, 32-case switch statement for routing, 45-item `useEffect` dependency array, all boot logic centralized. | Any panel change causes full page rebuild; boot sequence is opaque and untestable. |
| F2 | Giant panel components | **High** | `agent-detail-tabs.tsx` (2,939 lines), `office-panel.tsx` (2,410 lines), `task-board-panel.tsx` (2,206 lines) | Components with 2,000–3,000 lines mixing data fetching, business logic, and rendering. | Unmaintainable, untestable, and cause full re-renders on any state change. |
| F3 | 594 direct `fetch()` calls in components | **High** | `src/components/panels/*` | Components make `fetch()` calls directly with no shared API client, no retry logic, no request deduplication, no caching layer. | Network errors unhandled consistently; no loading/error states in ~50 panels. |
| F4 | Sub-agent UI with no backend | **Critical** | `agent-detail-tabs.tsx` | UI renders sub-agent configuration panel. Submitting it calls nothing functional. | Misleading UX. User changes saved nowhere. |
| F5 | 39+ `any` types in single component | **High** | `agent-detail-tabs.tsx` | Type safety entirely absent in the largest UI component. | Silent runtime failures; no IDE support. |
| F6 | No code splitting or lazy loading | **Medium** | `src/app/[[...panel]]/page.tsx` | All 43 panels eagerly imported. No `dynamic()` imports. | Large initial JS bundle; slow first load. |
| F7 | Accessibility gaps | **Medium** | Across `src/components/` | Only 13.6% of components use ARIA attributes. No keyboard navigation audit. No focus management on modal open/close. | Inaccessible to screen reader users; may violate WCAG 2.1 AA. |
| F8 | 19 instances of CSS `display:none` dead UI | **Low** | `src/components/panels/*` | Feature areas hidden via CSS rather than removed. Suggests incomplete or abandoned features. | Dead code in production; confusing for future developers. |
| F9 | Nav rail is 1,440 lines | **Medium** | `src/components/layout/nav-rail.tsx` | Navigation component has grown to layout, state, routing, and animation logic all mixed together. | Major maintenance burden; any nav change is risky. |
| F10 | Missing loading/error states | **Medium** | 50+ panel components | Panels fetch data but render nothing (or broken UI) while loading or on error. | Users see blank panels with no feedback during fetch. |

---

### D3. Backend / API

| # | Title | Severity | File(s) | Issue | Impact |
|---|-------|----------|---------|-------|--------|
| B1 | JSON.parse without try/catch | **High** | `webhooks/route.ts:59`, `workflows/route.ts:41,113,175`, `chat/messages/route.ts:44,73` | Multiple routes parse JSON from database fields without error handling. A single malformed DB record throws an unhandled exception. | Malformed data causes 500 errors on entire list endpoints; DoS via data corruption. |
| B2 | Excessive `any[]` in query results | **Medium** | `search/route.ts` (8 instances), `chat/messages/route.ts`, `audit/route.ts` | Database query results cast to `any[]` throughout API routes. | Schema changes silently break API responses; no compile-time safety. |
| B3 | Injection guard not universal | **Low** | `src/lib/injection-guard.ts` | Injection guard applied only to `task_prompt` in workflow creation. Agent souls, task descriptions, and comments are not scanned. | User-supplied text reaching AI agents bypasses safety checks. |
| B4 | OpenAPI spec cached at startup | **Low** | `src/app/api/docs/route.ts:9` | `openapi.json` read once at startup and cached indefinitely. No error handling for missing/corrupted file. | Stale docs post-deployment; startup crash if file absent. |
| B5 | Duplicate task title check blocks workflows | **Medium** | `src/app/api/tasks/route.ts` | Unique constraint on task title prevents legitimate workflow patterns where same-named tasks recur across different agents or projects. | Operational friction; workaround required for automation. |
| B6 | Rate limiting not distributed | **Medium** | `src/lib/rate-limit.ts` | In-memory rate limiter resets on process restart. Not safe for multi-process or multi-container deployments. | Rate limits bypassable by restarting the process. |
| B7 | `/api/diagnostics` accessible to `viewer` role | **High** | `src/app/api/diagnostics/route.ts` | Diagnostics endpoint exposes system configuration details. Should require `admin` role. | Information disclosure to low-privilege users. |

---

### D4. State Management

| # | Title | Severity | File(s) | Issue | Impact |
|---|-------|----------|---------|-------|--------|
| S1 | 1,128-line monolithic Zustand store | **High** | `src/store/index.ts` | Single store file with 40+ properties covering agents, tasks, UI state, connection status, notifications, etc. | Impossible to understand store shape at a glance; any change risks unintended side effects. |
| S2 | Global state overuse | **Medium** | `src/store/index.ts` (88 calls from components) | Component-local UI state stored in global Zustand, causing unnecessary re-renders across unrelated components. | Performance degradation; tight coupling between unrelated UI sections. |
| S3 | No memoization | **Medium** | `src/components/panels/*` | No `useMemo`, `useCallback`, or `React.memo` usage detected across panels. | Expensive re-renders on any state update in the 1,128-line global store. |

---

### D5. Realtime / WebSocket / Gateway

| # | Title | Severity | File(s) | Issue | Impact |
|---|-------|----------|---------|-------|--------|
| R1 | Event sequence gaps cause silent data loss | **High** | `src/lib/websocket.ts:475-481` | When gateway event sequence numbers show a gap, a warning is logged but no re-sync or recovery is triggered. Missed events are silently dropped. | Agent status, session state, cron state can silently drift from reality. Operator has no indication. |
| R2 | Heartbeat reconnect cycling race | **High** | `src/lib/websocket.ts:149-191` | After 3 missed pongs, `ws.close(4000)` is called immediately and reconnection begins. No check for existing pending connection. Two concurrent WebSocket connections can exist briefly. | Connection thrashing; potential gateway-side DoS; user sees constant reconnection toasts. |
| R3 | Three data sources with no conflict resolution | **Medium** | `websocket.ts`, `use-server-events.ts`, `use-smart-poll.ts` | WebSocket, SSE, and polling all call `updateTask()` unconditionally. Last-write-wins with no timestamp ordering. | Task status can revert to stale state when a slow polling response arrives after a faster SSE update. |
| R4 | SSE silent failure after 20 reconnects | **Medium** | `src/lib/use-server-events.ts:22-92` | After 20 failed reconnections (~40+ minutes), SSE permanently stops with no user notification. If WebSocket is also down, UI is completely stale with no indicator. | Operators work on stale data with no awareness. |
| R5 | Passive heartbeat mode allows zombie connections | **Medium** | `src/lib/websocket.ts:401-408` | If gateway doesn't support ping RPC, client switches to passive mode. A true network outage goes undetected indefinitely since no active probe is sent. | Gateway appears connected when it is not. |
| R6 | `pauseWhenDisconnected` creates dead UI | **Medium** | `src/lib/use-smart-poll.ts:58-65` | Components configured with `pauseWhenDisconnected: true` stop polling when WebSocket drops. If SSE also fails, UI is frozen on stale data with no fallback. | Silent data freeze during simultaneous multi-channel failure. |
| R7 | Device identity silent security downgrade | **Medium** | `src/lib/websocket.ts:215-261` | If gateway rejects device signature, code silently clears device identity and reconnects with token-only auth. User is not informed of the downgrade. | Operator unaware their session security was reduced. |
| R8 | Activity feed truncated at 1,000 items silently | **Low** | `src/store/index.ts:1018` | Activity array sliced to 1,000 with no notification or persistence fallback. | Long-running sessions lose audit trail beyond 1,000 events in-memory. |

---

### D6. Security

| # | Title | Severity | File(s) | Issue | Impact |
|---|-------|----------|---------|-------|--------|
| SEC1 | `.env.test` with real credentials committed | **Critical** | `.env.test` | Plain-text `AUTH_PASS=testpass1234!`, `API_KEY=test-api-key-e2e-12345`, `MC_ALLOW_ANY_HOST=1` committed to repository. | Any repository clone exposes usable credentials. Attack surface for staging/production environments using similar passwords. |
| SEC2 | Session cookie missing `__Host-` prefix | **High** | `src/lib/session-cookie.ts:3-15` | Cookie named `mc-session` without `__Host-` prefix. TODO comment acknowledges this but not fixed. | Subdomain cookie fixation attacks possible. Cookies can be set from insecure contexts. |
| SEC3 | API key placeholder in `.env.example` | **Medium** | `.env.example:14` | Default `API_KEY=generate-a-random-key` is a descriptive placeholder, not enforced as random at startup. | Deployments that copy `.env.example` without modification have a guessable API key. |
| SEC4 | `AUTH_PASS` falls back to plaintext if base64 invalid | **Medium** | `src/lib/db.ts:91-100` | `AUTH_PASS_B64` is preferred but code silently falls back to `AUTH_PASS` if base64 decode fails. No warning emitted. | Operators may believe they are using the secure path while actually on the less-safe path. |
| SEC5 | Diagnostics endpoint exposes config to `viewer` role | **High** | `src/app/api/diagnostics/route.ts` | Endpoint reveals system configuration details. Should require `admin`, not `viewer`. | Information disclosure to any authenticated user. |
| SEC6 | Path traversal risk in session recovery | **High** | `src/lib/openclaw-doctor-fix.ts:40-70` | File operations without full path resolution + bounds check. User-controlled path fragments could escape expected directory. | Directory traversal allowing reads/writes outside intended scope. |
| SEC7 | CSP `connect-src` too broad | **Medium** | `src/proxy.ts:86-95` | `connect-src http://127.0.0.1:*` and `http://localhost:*` allow connections to any localhost port. | XSS escalation via internal service probing; SSRF via script. |
| SEC8 | OAuth state token not validated | **Medium** | `src/app/api/auth/google/route.ts` | Google OAuth callback does not validate `state` parameter against server-generated value. | CSRF on OAuth callback; account linkage hijacking. |
| SEC9 | CSV injection in audit exports | **Medium** | Audit export routes | Audit log values not escaped for CSV formula injection (`=`, `+`, `-`, `@` prefixes). | Formula injection when exported files opened in spreadsheet applications. |
| SEC10 | Rate limit disableable via env var | **Low** | `src/lib/rate-limit.ts` | `MC_DISABLE_RATE_LIMIT=1` bypasses all rate limiting. No log warning when active. | Silent security downgrade in environments where this is set for testing and forgotten. |
| SEC11 | Hardcoded test credentials in Playwright config | **Medium** | `playwright.config.ts:33` | `API_KEY='test-api-key-e2e-12345'` hardcoded. Should read from environment. | Test credentials in source control; inflexible for different test environments. |
| SEC12 | Access request endpoint has no domain validation | **Low** | `src/app/api/auth/google/route.ts:58-63` | Anyone can submit an access request; no email domain allowlist. | Spam/DoS of access request queue; social engineering admins. |

---

### D7. Performance

| # | Title | Severity | File(s) | Issue | Impact |
|---|-------|----------|---------|-------|--------|
| P1 | No code splitting | **Medium** | `src/app/[[...panel]]/page.tsx` | All 43 panels eagerly imported on every page load. No `next/dynamic` usage. | Large initial JS bundle; slow Time-to-Interactive. |
| P2 | No memoization in panel components | **Medium** | `src/components/panels/*` | Heavy panel components re-render on any global store change with no memoization. | UI jank during high-frequency realtime updates. |
| P3 | SQLite WAL checkpoint not configured | **Low** | `src/lib/db.ts:27-30` | `cache_size = 1000` pages (~4MB). No explicit WAL checkpoint strategy. Auto-checkpoint at 1000 pages default. | Write stalls during auto-checkpoint; WAL file growth on busy instances. |
| P4 | No backpressure on SSE stream | **Low** | `src/app/api/events/route.ts` | EventSource has no rate limiting. A high-frequency event source will push all events to all connected browsers. | Browser CPU spike on rapid event sequences (e.g., bulk task imports). |

---

### D8. Testing

| # | Title | Severity | File(s) | Issue | Impact |
|---|-------|----------|---------|-------|--------|
| T1 | Coverage threshold at 60% | **Medium** | `vitest.config.ts` | Coverage threshold set below industry standard (80%+). Applies only to `src/lib/`; API routes and components have zero coverage targets. | Regressions in API and UI layers have no automated detection. |
| T2 | 164 TypeScript suppressions | **High** | `src/lib/*.ts`, `src/app/api/*.ts` | `@ts-ignore` and `@ts-nocheck` scattered across production code files. Strict mode is declared but not honored. | Type errors in suppressed locations are invisible; runtime failures hide behind compile-time silence. |
| T3 | React 19 ESLint rules disabled | **Medium** | `eslint.config.mjs:15-20` | Three React 19 hook rules disabled with comment "false positives… until refactor pass". | Component state/effect bugs in the affected patterns go undetected. |
| T4 | Timing-dependent E2E tests | **Low** | `tests/rate-limiting.spec.ts` | Rate limiting tests rely on real-time timing assumptions. Slow CI runners may cause flakiness. | Intermittent CI failures unrelated to actual bugs. |

---

### D9. DevEx / Build / Tooling

| # | Title | Severity | File(s) | Issue | Impact |
|---|-------|----------|---------|-------|--------|
| D1 | Inline health check script in Dockerfile | **Low** | `Dockerfile:40-46` | Health check is inline Node.js code rather than a separate script. | Hard to test or modify health check in isolation. |
| D2 | No API versioning in openapi.json | **Medium** | `openapi.json` | No version prefix in paths. No deprecation strategy documented. | Breaking changes to 131 routes cannot be introduced gracefully. |
| D3 | openapi.json manually maintained | **Low** | `openapi.json` (7,763 lines) | Spec is not auto-generated from TypeScript types or route handlers. Divergence between spec and implementation is guaranteed over time. | Documentation trust erodes; API clients break silently. |

---

### D10. Documentation

| # | Title | Severity | File(s) | Issue | Impact |
|---|-------|----------|---------|-------|--------|
| DOC1 | Sub-agent docs exist for unimplemented feature | **Medium** | `SKILL.md`, `docs/` | Documentation describes sub-agent capabilities that are not implemented. | Misleads integrators; creates support burden. |
| DOC2 | Framework adapter docs describe stubs as working | **Medium** | `README.md`, `docs/` | CrewAI, LangGraph, AutoGen, Claude SDK adapters documented as supported. They are non-functional stubs. | Integrators will spend time on integrations that cannot work. |
| DOC3 | Some docs reference future dates as "completed" | **Low** | Various docs | Dates like 2026-03-11 used in completed changelog entries that appear templated. | Minor credibility concern. |

---

### D11. Product / UX Consistency

| # | Title | Severity | File(s) | Issue | Impact |
|---|-------|----------|---------|-------|--------|
| UX1 | Cron management is read-only | **High** | `src/app/api/cron/route.ts`, cron UI panel | The cron panel shows scheduled jobs from OpenClaw but cannot create, edit, or delete them. Users have no indication the UI is read-only. | Operators attempt to manage cron from Mission Control and fail silently. |
| UX2 | Activity feed does not log agent lifecycle events | **Medium** | `src/app/api/activities/route.ts` | UI actions create activity records; adapter-side agent events (agent started, task completed, etc.) do not. | Activity feed shows only what the human did, not what the agent did. Core value prop missing. |
| UX3 | Workflow execution shows no execution history | **High** | Workflow UI panels | Workflows can be created but never run (engine absent). No execution history, no status, no result. | The workflows section of the UI is entirely decorative. |
| UX4 | Pipeline UI promises features not delivered | **Critical** | `src/app/api/pipelines/` | Pipelines endpoints return nothing. Any UI referencing pipelines is broken. | User-facing broken feature with no error message. |

---

## E. Top Priority Issues

The 10 most important issues to address before major feature expansion:

| Priority | Issue | Severity | Why First |
|----------|-------|----------|-----------|
| 1 | Remove/vault `.env.test` with committed credentials | Critical | Active security risk in repository |
| 2 | Implement task execution engine | Critical | Core product value absent |
| 3 | Implement or remove pipeline/sub-agent stubs | Critical | Broken API contracts mislead users and integrators |
| 4 | Event sequence gap recovery in WebSocket | High | Silent data loss corrupts operator view of reality |
| 5 | Split monolithic Zustand store | High | Every feature addition increases global re-render surface |
| 6 | Add `__Host-` prefix to session cookie | High | Cookie security downgrade vector |
| 7 | Restrict `/api/diagnostics` to `admin` role | High | Information disclosure to any authenticated user |
| 8 | Add path bounds check in `openclaw-doctor-fix.ts` | High | Path traversal vulnerability |
| 9 | Wrap all `JSON.parse()` in try/catch in API routes | High | Single malformed DB record crashes list endpoints |
| 10 | Audit and remediate 164 TypeScript suppressions | High | Strict mode is declared but not enforced; hides real bugs |

---

## F. Technical Debt Hotspots

Files or modules most likely to become expensive bottlenecks:

| File | Lines | Problem | Risk |
|------|-------|---------|------|
| `src/app/[[...panel]]/page.tsx` | 567 | All routing, boot, 43 imports, 45-item effect | Any panel work requires touching this file |
| `src/store/index.ts` | 1,128 | Monolithic global state, 40+ properties | Performance and coupling grow with every feature |
| `src/components/panels/agent-detail-tabs.tsx` | 2,939 | Data fetch, business logic, and render mixed; 39+ `any` | Untestable; any agent feature changes this file |
| `src/lib/websocket.ts` | 867 | 15+ mutable refs as state machine; no formal state transitions | Connection bugs are hard to reproduce and fix |
| `src/lib/migrations.ts` | 1,270 | Forward-only; no rollback; monolithic execution | A single bad migration is catastrophic |
| `src/lib/db.ts` | 572 | Inline schema seeding, password validation, admin user creation | Security-critical code mixed with infrastructure |
| `src/components/layout/nav-rail.tsx` | 1,440 | Navigation + state + routing + animation | Any nav change is risky |
| `src/app/api/tasks/route.ts` | 413 | All task CRUD + business logic in one route file | No service layer; grows with every task feature |

---

## G. Production Readiness Assessment

### What Is Missing Before Production

| Category | Status | Blocker? |
|----------|--------|---------|
| Task execution engine | ❌ Absent | **Yes** — core function |
| Workflow execution | ❌ Absent | **Yes** — core function |
| Sub-agent management | ❌ 5% complete | **Yes** for advertised feature |
| `.env.test` credential removal | ❌ Not done | **Yes** — security |
| `__Host-` session cookie | ❌ TODO comment only | Yes |
| Diagnostics RBAC fix | ❌ Not done | Yes |
| Path traversal fix | ❌ Not done | Yes |
| OAuth state token validation | ❌ Not done | Yes |
| Multi-process SQLite strategy | ⚠️ Documented? | No (single container fine) |
| Distributed rate limiting | ⚠️ Single-process only | No (single instance fine) |
| WebSocket sequence gap recovery | ❌ Logged, not recovered | Operational risk |
| Activity feed agent events | ❌ Absent | Core value prop |
| TypeScript `any` audit | ❌ 164 suppressions | Medium risk |
| API versioning | ❌ Absent | Long-term risk |
| Code splitting | ❌ Absent | Performance |
| Repository/service layer | ❌ Absent | Maintainability |

### Summary

The application **can run in production** in the sense that it starts, serves the UI, authenticates users, and stores data. But as an orchestration platform, it is not production-ready because it cannot orchestrate anything. It is a monitoring and metadata UI that shows you what tasks and agents exist, but cannot dispatch, execute, or coordinate work autonomously.

---

## H. Recommended Next Phase

A proposed remediation order. **Do not start new feature work until the first block is addressed.**

### Block 1 — Security Remediation (Sprint 1, non-negotiable)
1. Remove or vault `.env.test`; rotate any credentials that were committed
2. Implement `__Host-` prefix migration for session cookie
3. Restrict `/api/diagnostics` to `admin` role
4. Add path bounds check in `openclaw-doctor-fix.ts`
5. Add OAuth state token CSRF validation
6. Escape CSV exports against formula injection

### Block 2 — Broken Contract Fixes (Sprint 1-2)
7. Implement or clearly stub/404 the pipelines endpoints (do not silently return nothing)
8. Implement or remove sub-agent UI sections (no deceptive UX)
9. Mark non-OpenClaw framework adapters as `not yet implemented` in docs and UI

### Block 3 — Core Orchestration Engine (Sprints 2-5)
10. Design the task execution lifecycle (claim → setup → execute → capture result → report)
11. Implement task dispatch from queue to agents
12. Implement workflow execution engine (instantiate templates, track state)
13. Implement sub-agent lifecycle (create, assign task, track, report back)
14. Wire activity feed to agent lifecycle events (not just UI actions)
15. Implement cron job write operations (create/edit/delete, not read-only)

### Block 4 — Structural Cleanup (Sprints 3-6, in parallel with Block 3)
16. Wrap all `JSON.parse()` calls in try/catch across API routes
17. Audit and remediate TypeScript suppressions (target: reduce from 164 to <20)
18. Split Zustand store into domain-specific sub-stores
19. Extract a repository/DAO layer for database access
20. Break down `agent-detail-tabs.tsx`, `office-panel.tsx`, `task-board-panel.tsx`
21. Implement code splitting / lazy loading for the 43 panels
22. Implement WebSocket event sequence gap recovery

### Block 5 — Quality & Observability (Ongoing)
23. Raise test coverage thresholds to 80%; extend coverage to `src/app/` and `src/components/`
24. Re-enable disabled React 19 ESLint rules
25. Implement OpenAPI auto-generation from route handlers
26. Add WAL checkpoint monitoring / configuration
27. Implement `__Host-` cookie prefix fully
28. Add distributed rate limiting (Redis) if multi-process deployment is planned

---

## Quick Wins

Issues that can be fixed in under 30 minutes each with zero architectural risk:

- [ ] `JSON.parse` wrap in `src/app/api/webhooks/route.ts:59` — one try/catch
- [ ] `requireRole(request, 'admin')` on `/api/diagnostics`
- [ ] Remove `MC_ALLOW_ANY_HOST=1` from `.env.test`; add to `.gitignore`
- [ ] Add `not yet implemented` response to `pipelines/run` instead of empty 200
- [ ] `LOG_LEVEL=warn` when `MC_DISABLE_RATE_LIMIT=1` is set
- [ ] Add `path.resolve()` + bounds check in `openclaw-doctor-fix.ts`
- [ ] Move hardcoded `API_KEY` in `playwright.config.ts` to `process.env` with fallback

---

## Top 10 Issues (Condensed Reference)

| Rank | Issue | Severity |
|------|-------|----------|
| 1 | `.env.test` commits real credentials to repo | **Critical** |
| 2 | No task/workflow execution engine — the core product is unimplemented | **Critical** |
| 3 | Pipeline endpoints are empty stubs; sub-agent feature is 5% complete | **Critical** |
| 4 | WebSocket event sequence gaps cause silent state drift | **High** |
| 5 | Session cookie lacks `__Host-` prefix hardening (TODO unfulfilled) | **High** |
| 6 | `/api/diagnostics` leaks config to `viewer` role | **High** |
| 7 | Path traversal risk in `openclaw-doctor-fix.ts` | **High** |
| 8 | `JSON.parse` without error handling causes 500s on malformed DB data | **High** |
| 9 | 164 `@ts-ignore` suppressions undermine declared strict TypeScript policy | **High** |
| 10 | Monolithic 2,900-line components with mixed concerns and `any` types | **High** |

---

## Foundational Refactors to Consider Later

These are architectural improvements that are not urgent but will pay compounding dividends as the codebase scales:

1. **Repository/DAO layer** — `lib/repositories/agents.ts`, `lib/repositories/tasks.ts`, etc. Eliminates 131 files of inline SQL.
2. **Service layer** — `lib/services/task-dispatch.ts`, `lib/services/workflow-engine.ts`. Business logic out of route handlers.
3. **Formal WebSocket state machine** — Replace 15 mutable refs with a proper FSM (XState or manual). Eliminates a class of hard-to-reproduce connection bugs.
4. **Domain-split Zustand stores** — `agentStore`, `taskStore`, `uiStore`, `connectionStore`. Eliminates global re-renders.
5. **Panel lazy loading** — `next/dynamic` for all 43 panels. Reduces initial bundle by 60-70%.
6. **OpenAPI code generation** — Generate types and client from `openapi.json` using `openapi-typescript`. Eliminates `any[]` in API responses and keeps spec in sync.
7. **SQLite → PostgreSQL migration path** — Not needed now, but worth designing the abstraction layer (via a repository layer) so the migration is possible without rewriting 131 routes.
8. **Soft deletes** — Add `deleted_at` to all tables. Required for proper audit trails in a system that manages agents and tasks.

---

## Final Verdict

> **"Continue cautiously with targeted cleanup"**

Mission Control has the bones of a serious production system: professional security architecture, a comprehensive API, solid DevEx, and meaningful test coverage. The engineering team clearly knows what they're doing.

But the orchestration engine — the actual execution of tasks, workflows, and sub-agents — does not exist yet. The current system is a sophisticated dashboard for viewing and managing metadata about work that agents must perform externally. Before expanding the UI further, the execution layer needs to be designed and built. Everything else is paint on a house without plumbing.
