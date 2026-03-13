# React Router v6 → v7 Migration — GitHub Copilot Agent Kit

A set of **GitHub Copilot agents and skills** designed to automate safe, incremental React Router v6 → v7 migrations on large codebases (~1000 files). Each migration phase ships as its own PR, verified by automated checks and browser smoke tests — with zero breaking changes if the steps are followed in order.

---

## Table of Contents

- [What This Repository Is](#what-this-repository-is)
- [Quick Start](#quick-start)
- [Setup](#setup)
- [Agents](#agents)
  - [@rr-migrate — Execute migration steps](#rr-migrate--execute-migration-steps)
  - [@rr-verify — Validate each step](#rr-verify--validate-each-step)
  - [@rr-rca — Investigate failures](#rr-rca--investigate-failures)
- [Skills](#skills)
- [Migration Workflow](#migration-workflow)
- [Repository Structure](#repository-structure)

---

## What This Repository Is

This repository is a **GitHub Copilot workspace extension** — a collection of agents, skills, and documentation files that you add to your own project so that Copilot Chat can assist with React Router migrations.

It is **not a runnable application**. It contains no source code, no `package.json`, and no build configuration. Everything here is Markdown-based instruction that Copilot reads and acts on.

**What it provides:**

| Component | Count | Purpose |
|-----------|-------|---------|
| Agents (`.agent.md`) | 3 | Orchestrate migration tasks in Copilot Chat |
| Skills (`SKILL.md` + references) | 5 | Deep knowledge loaded on demand |
| Copilot instructions | 1 | Context always available to Copilot |
| Migration guide | 1 | 600-line strategy with scripts and checklists |

**Key advantages over a manual migration:**
- ✅ Zero breaking changes when phases are done in order
- ✅ Each phase ships as an independent PR (≤ 50 files, except the automated import rewrite)
- ✅ 90% of import rewrites automated via `sed`
- ✅ Each step verified with build checks, test runs, and browser smoke tests
- ✅ Rollback plan included for every phase

---

## Quick Start

```
# In VS Code Copilot Chat, after setup:

@rr-migrate enable v7_startTransition flag

@rr-verify verify v7_startTransition flag

@rr-rca build fails after import rewrite
```

---

## Setup

### 1. Copy this repository's files into your project

Copy the following directories from this repository into your project's `.github/` folder:

```
your-project/
└── .github/
    ├── agents/           ← from this repo's agents/
    ├── docs/             ← from this repo's docs/
    ├── instructions/     ← from this repo's instructions/
    └── skills/           ← from this repo's skills/
```

> **Why `.github/`?** GitHub Copilot automatically discovers agents (`.agent.md`), instructions (`.instructions.md`), and prompt files placed inside `.github/`. Placing the files there means Copilot picks them up without any additional configuration.

### 2. Enable GitHub Copilot in VS Code

Ensure you have the **GitHub Copilot** and **GitHub Copilot Chat** extensions installed and signed in.

### 3. Open Copilot Chat

Open the Copilot Chat panel (`Ctrl+Alt+I` / `⌃⌥I`) and start a conversation with one of the agents.

### 4. Verify agent discovery

Type `@` in the Copilot Chat input — you should see `@rr-migrate`, `@rr-verify`, and `@rr-rca` in the autocomplete list.

---

## Agents

Agents are defined in the `agents/` directory as `.agent.md` files. Each agent has a specific role and a fixed set of tools it may use.

---

### `@rr-migrate` — Execute migration steps

**File:** `agents/rr-migrate.agent.md`

**Role:** Executes a single migration phase: reads the migration guide, runs an impact assessment, creates a todo list, applies changes file-by-file, runs the build, and reports the result.

**Available tools:** `read`, `edit`, `search`, `execute`, `todo`

**Usage examples:**

```
@rr-migrate enable v7_startTransition flag
@rr-migrate enable v7_relativeSplatPath flag
@rr-migrate enable v7_fetcherPersist flag
@rr-migrate enable v7_normalizeFormMethod flag
@rr-migrate enable v7_partialHydration flag
@rr-migrate enable v7_skipActionErrorRevalidation flag
@rr-migrate upgrade package from react-router-dom to react-router@7
@rr-migrate rewrite all imports from react-router-dom to react-router
@rr-migrate remove json() usage from loaders     # replaces json() with data()
@rr-migrate remove defer() usage from loaders    # replaces defer() with data()
@rr-migrate remove future flags from router config
```

**What it does for each step:**

1. Reads `MIGRATION_GUIDE_v6_to_v7.md` for the phase-specific strategy
2. Runs an impact assessment (counts affected files, reports patterns)
3. Creates a todo list tracking every file to change
4. Applies the minimal required change to each file
5. Runs `npm run build` and `npm test` to verify
6. Reports a summary and suggests `@rr-verify` as the next step

**Output format:**

```
## Migration Step Complete: [step name]

**Files changed:** X
**Pattern:** `before` → `after`
**Build status:** ✅ passing / ❌ failing
**Test status:** ✅ passing / ❌ failing / ⚠️ not run

### Files requiring manual review:
- [file](path) — reason

### Next step:
Run `@rr-verify` to validate, then proceed to [next phase].
```

**Constraints:**
- Never changes multiple phases in one session
- Never skips the impact assessment
- Never modifies test files and source files in the same commit

---

### `@rr-verify` — Validate each step

**File:** `agents/rr-verify.agent.md`

**Role:** Read-only validation agent. Checks that a completed migration step is correct, complete, and free of regressions. Never edits files.

**Available tools:** `read`, `search`, `execute`, `todo`

**Usage examples:**

```
@rr-verify verify v7_startTransition flag
@rr-verify verify import rewrite
@rr-verify verify package upgrade
@rr-verify verify deprecation cleanup
@rr-verify full smoke test
```

**What it checks (per phase):**

| Phase | Checks |
|-------|--------|
| Future flag | Config file has flag set to `true`; no leftover old patterns; build ✅; tests ✅ |
| Package upgrade | `package.json` has `react-router`, NOT `react-router-dom`; lock file correct; `npm ls` output |
| Import rewrite | No remaining `react-router-dom` imports; `RouterProvider` imports from `react-router/dom`; no broken imports |
| Deprecation cleanup | No `json(` or `defer(` calls; loader return types are valid |
| Browser smoke test | Launches Chrome via `browser-verify` skill, navigates routes, checks console errors, takes screenshots |

**Output format:**

```
## Verification Report: [phase name]

**Status:** ✅ PASS / ❌ FAIL / ⚠️ PASS WITH WARNINGS

### Checks Performed
| Check | Result | Details |
|-------|--------|---------|
| Config flag present | ✅ | ... |
| Build succeeds     | ✅ | ... |

### BLOCKING Issues (must fix before proceeding)
### Warnings (can fix later)
### Recommendation
```

---

### `@rr-rca` — Investigate failures

**File:** `agents/rr-rca.agent.md`

**Role:** Root cause analysis agent. Investigates migration failures and provides fix recommendations. Does not auto-fix — reports findings and suggests targeted changes.

**Available tools:** `read`, `search`, `execute`, `todo`

**Usage examples:**

```
@rr-rca build fails after import rewrite
@rr-rca blank page after upgrading to react-router@7
@rr-rca tests failing after enabling v7_normalizeFormMethod
@rr-rca routes returning 404 after migration
@rr-rca TypeScript errors after removing json()
```

**Error categories it handles:**

| Category | Examples |
|----------|---------|
| Compile-time | TypeScript errors, missing exports, broken imports |
| Test regressions | Snapshot failures, mock mismatches after API change |
| Runtime | Blank page, wrong routes rendered, data not loading |
| Routing | 404s, relative path resolution changes |
| Deprecation warnings | Remaining `json()`, `defer()`, `fallbackElement` usage |

**Investigation procedure:**

1. Classifies the error type from symptoms
2. Gathers evidence (git diff, error messages, affected files)
3. Traces to root cause (migration-caused vs pre-existing)
4. Reports findings with exact file paths and line numbers
5. Provides targeted fix recommendations for `@rr-migrate`

---

## Skills

Skills are collections of reference knowledge loaded by agents on demand. They live in the `skills/` directory.

### `browser-verify`

**Path:** `skills/browser-verify/`

Chrome browser automation via the Chrome DevTools Protocol (CDP). Requires no npm dependencies — uses Node 22 native WebSocket and a shell script to launch Chrome.

**Use cases:** Smoke-testing routes after migration, taking screenshots, verifying no console errors, checking network requests.

**50+ commands including:**

| Category | Commands |
|----------|---------|
| Navigation | `navigate`, `reload`, `back`, `forward`, `page-info`, `new-tab` |
| Visual | `screenshot`, `viewport`, `pdf`, `emulate` |
| DOM | `evaluate`, `get-html`, `get-text`, `query-all`, `get-attribute` |
| Interaction | `click`, `type`, `press`, `hover`, `select`, `check`, `drag`, `upload` |
| Waiting | `wait`, `wait-visible`, `wait-text`, `wait-url`, `wait-network-idle`, `sleep` |
| Debugging | `get-console`, `get-network`, `get-cookies` |

---

### `react-router-code-review`

**Path:** `skills/react-router-code-review/`

A 10-item review checklist for React Router patterns, with guidance on valid vs. invalid patterns and context-sensitive rules.

**Checks include:** data loading via loaders not `useEffect`, type-safe route params, `<Form>` / `useFetcher` for mutations, error boundaries with `errorElement`, pending states via `useNavigation`.

**References:** `data-loading.md`, `mutations.md`, `error-handling.md`, `navigation.md`

---

### `react-router-data-mode`

**Path:** `skills/react-router-data-mode/`

Patterns for the data mode API: `createBrowserRouter`, `RouterProvider`, loaders, actions, optimistic UI, and SSR.

**References:** `routing.md`, `route-object.md`, `data-loading.md`, `actions.md`, `navigation.md`, `pending-ui.md`, `ssr.md`

---

### `react-router-declarative-mode`

**Path:** `skills/react-router-declarative-mode/`

Patterns for the declarative API: `<BrowserRouter>`, `<Routes>`, `<Route>`, `<Link>`, `<NavLink>`, `useNavigate`, `useParams`, `useSearchParams`.

**References:** `routing.md`, `navigation.md`, `url-values.md`

---

### `frontend-react-router-best-practices`

**Path:** `skills/frontend-react-router-best-practices/`

55 performance and architecture rules across 11 categories, each in its own file with bad/good code examples.

| Category | Rules |
|----------|-------|
| Data loading | Avoid waterfalls, parallel fetches, request caching, colocated queries |
| Actions & forms | Validation, redirect after action, Zod transforms, pending state |
| Navigation | Link prefetching, avoid `navigate(-1)`, prefetch cache |
| Error handling | Route-level and layout-level error boundaries |
| Route organization | Naming, `shouldRevalidate`, resource routes, auth middleware |
| Middleware | Session, logging, server timing, request ID, batching |
| Migration (v6→v7) | `json()` → `data()`, `defer()` → `data()`, named action intent |
| Security | Fetch guards, CORS headers, client IP |
| Streaming | Await + Suspense, SSE event streams |
| TypeScript | Typed cookies, safe redirects |

---

## Migration Workflow

The migration is split into **13 pull requests**, each safe to ship independently:

| PR | Phase | Effort | Risk |
|----|-------|--------|------|
| 1 | Install latest react-router-dom@6 minor | Auto | 🟢 None |
| 2 | Enable `v7_startTransition` | Auto | 🟢 Low |
| 3 | Enable `v7_relativeSplatPath` + fix splat routes | Semi-auto | 🟡 Medium |
| 4 | Enable `v7_fetcherPersist` | Auto | 🟢 Low |
| 5 | Enable `v7_normalizeFormMethod` + fix comparisons | Auto | 🟢 Low |
| 6 | Enable `v7_partialHydration` | Manual | 🟡 Medium |
| 7 | Enable `v7_skipActionErrorRevalidation` + fix actions | Manual | 🟡 Medium |
| 8 | Swap `react-router-dom` → `react-router@7` | Auto (npm) | 🟢 Low |
| 9 | Rewrite all imports (~1000 files, automated sed) | Auto | 🟢 Low |
| 10 | Fix `RouterProvider` deep import (`react-router/dom`) | Manual | 🟢 Low |
| 11 | Remove `json()` usage | Semi-auto | 🟡 Medium |
| 12 | Remove `defer()` usage | Semi-auto | 🟡 Medium |
| 13 | Delete future flags config (now defaults in v7) | Manual | 🟢 Low |

**Typical session flow:**

```
1. @rr-migrate enable v7_startTransition flag
   → Agent assesses impact, applies changes, runs build/test

2. @rr-verify verify v7_startTransition flag
   → Agent checks config, searches for leftover patterns, runs build/test

3. (if issues) @rr-rca build fails after enabling v7_startTransition
   → Agent investigates root cause, recommends fix

4. Open PR, merge, repeat for next phase
```

For the full strategy, automation scripts, verification checklists, and rollback plan, see [`docs/MIGRATION_GUIDE_v6_to_v7.md`](docs/MIGRATION_GUIDE_v6_to_v7.md).

---

## Repository Structure

```
react-router-migration/
│
├── README.md                          ← This file
│
├── agents/
│   ├── rr-migrate.agent.md            ← @rr-migrate: execute migration steps
│   ├── rr-verify.agent.md             ← @rr-verify: validate each step
│   └── rr-rca.agent.md                ← @rr-rca: investigate failures
│
├── instructions/
│   └── react-router-v6-to-v7-migration.instructions.md
│                                      ← Always-on Copilot context
│
├── docs/
│   └── MIGRATION_GUIDE_v6_to_v7.md   ← Complete 600-line migration strategy
│
└── skills/
    ├── browser-verify/
    │   ├── SKILL.md                   ← Chrome CDP automation (50+ commands)
    │   ├── scripts/                   ← chrome-launcher.sh, cdp.mjs
    │   └── references/cdp-commands.md
    │
    ├── react-router-code-review/
    │   ├── SKILL.md                   ← 10-item code review checklist
    │   └── references/                ← data-loading, mutations, errors, navigation
    │
    ├── react-router-data-mode/
    │   ├── SKILL.md                   ← createBrowserRouter patterns
    │   └── references/                ← routing, loaders, actions, pending-ui, SSR
    │
    ├── react-router-declarative-mode/
    │   ├── SKILL.md                   ← BrowserRouter/Routes patterns
    │   └── references/                ← routing, navigation, url-values
    │
    └── frontend-react-router-best-practices/
        ├── SKILL.md                   ← 55 rules summary
        └── rules/                     ← 55 individual rule files
```
