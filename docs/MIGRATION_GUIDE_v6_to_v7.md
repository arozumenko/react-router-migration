# React Router v6 → v7 Migration Guidelines

> For a ~1000-file React application. This guide is structured for incremental, safe migration with zero downtime.

## Table of Contents

- [Copilot Agents](#copilot-agents)
- [Prerequisites](#prerequisites)
- [Migration Strategy Overview](#migration-strategy-overview)
- [Phase 1: Enable Future Flags (in v6)](#phase-1-enable-future-flags-in-v6)
- [Phase 2: Upgrade Package to v7](#phase-2-upgrade-package-to-v7)
- [Phase 3: Update Imports](#phase-3-update-imports)
- [Phase 4: Deprecation Cleanup](#phase-4-deprecation-cleanup)
- [Phase 5: (Optional) Adopt Framework Mode](#phase-5-optional-adopt-framework-mode)
- [Automation Scripts](#automation-scripts)
- [Verification Checklist](#verification-checklist)
- [Risk Assessment](#risk-assessment)
- [Rollback Plan](#rollback-plan)

---

## Copilot Agents

Three dedicated agents orchestrate this migration. Use them in VS Code Copilot Chat:

| Agent | Invoke | Purpose |
|-------|--------|---------|
| **Migration** | `@rr-migrate` | Executes migration steps — enables flags, rewrites imports, removes deprecations |
| **Verification** | `@rr-verify` | Validates each step — checks completeness, runs build/tests, browser smoke tests |
| **RCA** | `@rr-rca` | Investigates failures — traces build errors, runtime issues, test regressions to root cause |

### Workflow per Migration Step

```
1. @rr-migrate "enable v7_startTransition flag"    ← executes the change
2. @rr-verify "verify v7_startTransition flag"     ← validates it's correct
3. @rr-rca "build fails after v7_startTransition"  ← only if something broke
4. Commit, PR, ship
5. Repeat for next step
```

### Agent Handoffs

The agents are wired to delegate to each other:
- `@rr-migrate` → suggests `@rr-verify` after completing a step
- `@rr-verify` → suggests `@rr-rca` if blocking issues are found
- `@rr-rca` → suggests `@rr-migrate` with the fix to apply

### Skills Used by Agents

| Skill | Used By | Purpose |
|-------|---------|---------|
| `browser-verify` | `@rr-verify`, `@rr-rca` | Launch Chrome, check console, take screenshots |
| `react-router-code-review` | `@rr-migrate`, `@rr-verify`, `@rr-rca` | Review patterns, checklists |
| `react-router-data-mode` | `@rr-migrate`, `@rr-rca` | v7 data mode patterns |
| `react-router-declarative-mode` | `@rr-migrate`, `@rr-rca` | v7 declarative mode patterns |

---

## Prerequisites

| Requirement | Minimum Version |
|-------------|-----------------|
| Node.js | 20+ |
| React | 18+ |
| react-dom | 18+ |
| react-router-dom | 6.x (latest minor) |

**Before starting:** Update to the latest `react-router-dom@6` to get all future flags and deprecation warnings.

```bash
npm install react-router-dom@6
```

---

## Migration Strategy Overview

React Router v7 has **no breaking changes** if all future flags are enabled in v6 first. The migration is designed as:

```
Phase 1: Enable future flags one at a time (still on v6) — SHIP EACH FLAG
Phase 2: Swap package from react-router-dom to react-router (v7)
Phase 3: Update all imports
Phase 4: Clean up deprecated APIs (json, defer)
Phase 5: (Optional) Adopt Vite plugin / framework mode
```

**Critical principle for a 1000-file app:** Make one change at a time, commit, test, ship. Never batch multiple flags or changes into one PR.

---

## Phase 1: Enable Future Flags (in v6)

Each flag should be **a separate PR** that gets merged and deployed independently. This is the safest approach for a large codebase.

### Flag 1: `v7_relativeSplatPath`

**What it changes:** Relative path matching for multi-segment splat paths like `dashboard/*`.

**Risk level:** 🟡 Medium — requires code changes if you use splat routes with relative links.

**Enable the flag:**

```tsx
// If using <BrowserRouter>
<BrowserRouter future={{ v7_relativeSplatPath: true }}>

// If using createBrowserRouter
createBrowserRouter(routes, {
  future: { v7_relativeSplatPath: true },
});
```

**Required code changes:** Find all routes with `path="something/*"` that have relative `<Link>` children.

```bash
# Find affected routes
grep -rn 'path=.*\/\*' src/ --include='*.tsx' --include='*.jsx' --include='*.ts'
```

**Before:**
```tsx
<Route path="dashboard/*" element={<Dashboard />} />
```

**After:** Split into parent + child:
```tsx
<Route path="dashboard">
  <Route path="*" element={<Dashboard />} />
</Route>
```

And update relative links inside `<Dashboard>`:
```tsx
// Before
<Link to="team">Team</Link>

// After
<Link to="../team">Team</Link>
```

**How to verify:** Navigate to every splat route and confirm links work correctly.

---

### Flag 2: `v7_startTransition`

**What it changes:** Uses `React.useTransition` instead of `React.useState` for router state updates.

**Risk level:** 🟢 Low — usually no code changes needed.

**Enable the flag:**

```tsx
<BrowserRouter future={{ v7_startTransition: true }}>
// or
<RouterProvider future={{ v7_startTransition: true }}>
```

**Required code changes:** Only if you use `React.lazy` **inside** a component (not at module scope).

```bash
# Find React.lazy inside components (not module-level)
grep -rn 'React.lazy\|= lazy(' src/ --include='*.tsx' --include='*.jsx'
```

If found inside a function body, move it to module scope:

```tsx
// ❌ Inside component
function MyComponent() {
  const LazyChild = React.lazy(() => import('./Child'));
  ...
}

// ✅ Module scope
const LazyChild = React.lazy(() => import('./Child'));
function MyComponent() { ... }
```

---

### Flag 3: `v7_fetcherPersist`

> **Skip if NOT using `<RouterProvider>`** (i.e., you use `<BrowserRouter>` with `<Routes>`)

**What it changes:** Fetcher lifecycle is based on idle state, not component unmount.

**Risk level:** 🟢 Low — rarely requires changes.

```tsx
createBrowserRouter(routes, {
  future: { v7_fetcherPersist: true },
});
```

**Required code changes:** Check `useFetchers` usage — fetchers may persist longer.

```bash
grep -rn 'useFetchers\|useFetcher' src/ --include='*.tsx' --include='*.jsx' --include='*.ts'
```

---

### Flag 4: `v7_normalizeFormMethod`

> **Skip if NOT using `<RouterProvider>`**

**What it changes:** `formMethod` is now uppercase (`"POST"` not `"post"`).

**Risk level:** 🟡 Medium — requires updating all formMethod comparisons.

```tsx
createBrowserRouter(routes, {
  future: { v7_normalizeFormMethod: true },
});
```

**Required code changes:**

```bash
# Find all formMethod comparisons
grep -rn 'formMethod' src/ --include='*.tsx' --include='*.jsx' --include='*.ts'
```

```tsx
// Before
useNavigation().formMethod === "post"
useFetcher().formMethod === "get"

// After
useNavigation().formMethod === "POST"
useFetcher().formMethod === "GET"
```

---

### Flag 5: `v7_partialHydration`

> **Skip if NOT using `<RouterProvider>`**

**What it changes:** Enables partial hydration, primarily for SSR.

**Risk level:** 🟢 Low for SPA apps.

```tsx
createBrowserRouter(routes, {
  future: { v7_partialHydration: true },
});
```

**Required code changes:** Replace `fallbackElement` with `HydrateFallback` on the root route:

```tsx
// Before
<RouterProvider router={router} fallbackElement={<Fallback />} />

// After — move fallback to root route definition
{
  path: "/",
  Component: Layout,
  HydrateFallback: Fallback,
  // or: hydrateFallbackElement: <Fallback />,
  children: [],
}

<RouterProvider router={router} />
```

---

### Flag 6: `v7_skipActionErrorRevalidation`

> **Skip if NOT using `createBrowserRouter`**

**What it changes:** Loaders no longer revalidate after actions return 4xx/5xx.

**Risk level:** 🟡 Medium — if actions mutate data before throwing errors.

```tsx
createBrowserRouter(routes, {
  future: { v7_skipActionErrorRevalidation: true },
});
```

**Required code changes:** If any action mutates data before throwing an error, either:

1. Move validation before mutation:
```tsx
async function action() {
  if (detectError()) throw new Response(error, { status: 400 }); // validate first
  await mutateSomeData(); // then mutate
}
```

2. Or opt into revalidation via `shouldRevalidate`:
```tsx
function shouldRevalidate({ actionStatus, defaultShouldRevalidate }) {
  if (actionStatus != null && actionStatus >= 400) return true;
  return defaultShouldRevalidate;
}
```

---

## Phase 2: Upgrade Package to v7

Once **all** future flags are enabled and deployed:

```bash
npm uninstall react-router-dom
npm install react-router@latest
```

In `package.json`, you should only have `"react-router"` — not `"react-router-dom"`.

---

## Phase 3: Update Imports

This is the highest-volume change for a 1000-file app. Use the automated script.

### Automated Import Rewriting (macOS)

```bash
# Dry run — count files that will change
grep -rl 'from "react-router-dom"\|from '\''react-router-dom'\''' src/ --include='*.tsx' --include='*.ts' --include='*.jsx' --include='*.js' | wc -l

# Apply — rewrite all imports
find ./src \( -name "*.tsx" -o -name "*.ts" -o -name "*.js" -o -name "*.jsx" \) -type f \
  -exec sed -i '' 's|from "react-router-dom"|from "react-router"|g' {} + \
  -exec sed -i '' "s|from 'react-router-dom'|from 'react-router'|g" {} +
```

### DOM-Specific Imports (Manual)

Two components must come from `react-router/dom` because they depend on `react-dom`:

| Component | v6 Import | v7 Import |
|-----------|-----------|-----------|
| `RouterProvider` | `react-router-dom` | `react-router/dom` |
| `HydratedRouter` | `react-router-dom` | `react-router/dom` |

```bash
# Find RouterProvider imports and update to deep import
grep -rn "RouterProvider" src/ --include='*.tsx' --include='*.ts' -l
```

```tsx
// Before
import { RouterProvider } from "react-router-dom";

// After (in browser code)
import { RouterProvider } from "react-router/dom";

// After (in test code — non-DOM context)
import { RouterProvider } from "react-router";
```

### Everything Else Stays the Same

All other imports (`useNavigate`, `useParams`, `Link`, `NavLink`, `Outlet`, `useLoaderData`, `Form`, `useFetcher`, `redirect`, `useLocation`, `useSearchParams`, etc.) simply change from `react-router-dom` to `react-router`.

---

## Phase 4: Deprecation Cleanup

These are deprecated in v7 and will be removed in a future version. Clean them up now.

### Replace `json()` helper

```bash
grep -rn 'from.*react-router.*json\|return json(' src/ --include='*.tsx' --include='*.ts'
```

```tsx
// Before
import { json } from "react-router-dom";
export function loader() {
  return json({ data: "value" });
}

// After — just return the object
export function loader() {
  return { data: "value" };
}

// If you need explicit JSON serialization for non-serializable data:
export function loader() {
  return Response.json({ data: "value" });
}
```

### Replace `defer()` helper

```bash
grep -rn 'from.*react-router.*defer\|return defer(' src/ --include='*.tsx' --include='*.ts'
```

```tsx
// Before
import { defer } from "react-router-dom";
export function loader() {
  return defer({ lazy: fetchData() });
}

// After — return promises directly
export function loader() {
  return { lazy: fetchData() };
}
```

---

## Phase 5: (Optional) Adopt Framework Mode

This is a **separate, larger project** — do NOT combine with the v6→v7 upgrade. The steps above get you to v7 in "library mode" where everything works exactly as before.

Framework mode (Vite plugin) adds:
- File-based routing via `routes.ts`
- Automatic code-splitting per route
- SSR/SSG support
- Type-safe route modules

If you want to adopt it later, see the official guides:
- From `<BrowserRouter>`: https://reactrouter.com/upgrading/component-routes
- From `<RouterProvider>`: https://reactrouter.com/upgrading/router-provider

---

## Automation Scripts

### Script: Count impact by pattern

```bash
#!/bin/bash
# Run from project root to assess migration scope

echo "=== React Router v6→v7 Migration Impact Assessment ==="
echo ""

echo "--- Import Counts ---"
echo "Files importing from react-router-dom:"
grep -rl "from ['\"]react-router-dom['\"]" src/ --include='*.tsx' --include='*.ts' --include='*.jsx' --include='*.js' 2>/dev/null | wc -l

echo ""
echo "--- Splat Routes (Flag 1) ---"
grep -rn "path=.*\*" src/ --include='*.tsx' --include='*.jsx' --include='*.ts' 2>/dev/null | head -20

echo ""
echo "--- React.lazy Inside Components (Flag 2) ---"
grep -rn "React.lazy\|= lazy(" src/ --include='*.tsx' --include='*.jsx' 2>/dev/null | head -20

echo ""
echo "--- formMethod comparisons (Flag 4) ---"
grep -rn "formMethod.*===\|===.*formMethod" src/ --include='*.tsx' --include='*.ts' 2>/dev/null | head -20

echo ""
echo "--- json() usage (Deprecation) ---"
grep -rn "return json(" src/ --include='*.tsx' --include='*.ts' 2>/dev/null | wc -l

echo ""
echo "--- defer() usage (Deprecation) ---"
grep -rn "return defer(" src/ --include='*.tsx' --include='*.ts' 2>/dev/null | wc -l

echo ""
echo "--- RouterProvider usage (DOM import) ---"
grep -rn "RouterProvider" src/ --include='*.tsx' --include='*.ts' 2>/dev/null | head -10

echo ""
echo "--- fallbackElement usage (Flag 5) ---"
grep -rn "fallbackElement" src/ --include='*.tsx' --include='*.ts' 2>/dev/null | head -10

echo ""
echo "--- useFetchers/useFetcher usage (Flag 3) ---"
grep -rn "useFetcher" src/ --include='*.tsx' --include='*.ts' 2>/dev/null | wc -l
```

### Script: Bulk import rewrite (macOS)

```bash
#!/bin/bash
# Rewrites all react-router-dom imports to react-router
# Run with --dry-run first!

set -euo pipefail

DRY_RUN=false
if [[ "${1:-}" == "--dry-run" ]]; then
  DRY_RUN=true
fi

SRC_DIR="${2:-src}"

FILES=$(find "$SRC_DIR" \( -name "*.tsx" -o -name "*.ts" -o -name "*.js" -o -name "*.jsx" \) -type f)
CHANGED=0

for f in $FILES; do
  if grep -q "react-router-dom" "$f"; then
    if $DRY_RUN; then
      echo "WOULD CHANGE: $f"
      grep -n "react-router-dom" "$f"
    else
      sed -i '' 's|from "react-router-dom"|from "react-router"|g' "$f"
      sed -i '' "s|from 'react-router-dom'|from 'react-router'|g" "$f"
      echo "CHANGED: $f"
    fi
    CHANGED=$((CHANGED + 1))
  fi
done

echo ""
echo "Total files ${DRY_RUN:+would be }changed: $CHANGED"
```

---

## Verification Checklist

### After Each Future Flag

- [ ] App builds without errors (`npm run build`)
- [ ] All existing tests pass (`npm test`)
- [ ] Manual smoke test: navigate through major flows
- [ ] No console warnings/errors related to routing
- [ ] PR reviewed and merged independently

### After Package Upgrade (Phase 2)

- [ ] `package.json` has `react-router` only (no `react-router-dom`)
- [ ] `node_modules` reinstalled clean (`rm -rf node_modules && npm install`)
- [ ] TypeScript compiles without errors
- [ ] All tests pass

### After Import Rewrite (Phase 3)

- [ ] No remaining imports from `react-router-dom` in src/
- [ ] `RouterProvider` imported from `react-router/dom` (not `react-router`)
- [ ] Test files use `react-router` (non-DOM import OK for testing)
- [ ] Build succeeds
- [ ] All routes render correctly

### After Deprecation Cleanup (Phase 4)

- [ ] No `json()` imports/calls remain
- [ ] No `defer()` imports/calls remain
- [ ] Loader return types still work (TypeScript checks pass)

---

## Risk Assessment

| Change | Files Affected (est.) | Risk | Automated? |
|--------|----------------------|------|-----------|
| Future flag: `v7_relativeSplatPath` | 5-50 (splat routes) | 🟡 Medium | Manual review needed |
| Future flag: `v7_startTransition` | 0-5 (lazy in components) | 🟢 Low | Grep + manual |
| Future flag: `v7_fetcherPersist` | 0-10 | 🟢 Low | Grep |
| Future flag: `v7_normalizeFormMethod` | 5-30 | 🟡 Medium | Grep + sed |
| Future flag: `v7_partialHydration` | 1-3 | 🟢 Low | Manual |
| Future flag: `v7_skipActionErrorRevalidation` | 0-20 (actions with errors) | 🟡 Medium | Manual review |
| Package swap (v6→v7) | 1 (package.json) | 🟢 Low | npm command |
| Import rewrite | **All 1000 files** | 🟢 Low | **Fully automated (sed)** |
| `RouterProvider` deep import | 1-5 | 🟢 Low | Grep + manual |
| `json()` removal | 20-100 | 🟢 Low | Grep + sed |
| `defer()` removal | 5-30 | 🟢 Low | Grep + sed |

**Total estimated effort:** The bulk of files (import rewrite) is fully automated. Manual work is concentrated in ~50-100 files for flag-related changes.

---

## Rollback Plan

Each phase is independently reversible:

| Phase | Rollback |
|-------|----------|
| Future flags | Remove the flag from router config, revert code changes |
| Package upgrade | `npm install react-router-dom@6 && npm uninstall react-router` |
| Import rewrite | `git revert` the import change commit |
| Deprecation cleanup | `git revert` — original `json()`/`defer()` still work in v7 |

**Key safeguard:** Because each future flag is shipped as a separate PR, you can identify exactly which change caused a regression and revert only that flag.

---

## Recommended PR Sequence

```
PR 1:  npm install react-router-dom@6 (latest v6 minor)
PR 2:  Enable v7_startTransition flag
PR 3:  Enable v7_relativeSplatPath + fix splat routes
PR 4:  Enable v7_fetcherPersist (if using RouterProvider)
PR 5:  Enable v7_normalizeFormMethod + fix comparisons (if using RouterProvider)
PR 6:  Enable v7_partialHydration (if using RouterProvider)
PR 7:  Enable v7_skipActionErrorRevalidation + fix actions (if using RouterProvider)
PR 8:  Swap package: react-router-dom → react-router@7
PR 9:  Rewrite all imports (automated)
PR 10: Fix RouterProvider deep import (react-router/dom)
PR 11: Remove json() usage
PR 12: Remove defer() usage
PR 13: Delete future flags config (they're now defaults)
```

Each PR should be ≤ 50 files changed (except PR 9 which is fully automated). Ship and monitor each before proceeding.
