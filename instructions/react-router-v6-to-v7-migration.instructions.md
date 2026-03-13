---
description: "Use when migrating React Router v6 to v7, planning a migration, enabling future flags, rewriting imports from react-router-dom to react-router, replacing json/defer utilities, upgrading packages, or reviewing migration progress."
---

# React Router v6 → v7 Migration

This workspace is undergoing a React Router v6 → v7 migration (~1000 files).

## Key Facts

- v7 has **zero breaking changes** if all 6 future flags are enabled in v6 first
- `react-router-dom` is replaced by `react-router` (single package)
- Migration is split into 5 phases, each with its own PR

## Phases

1. **Future Flags** — enable all 6 flags one-by-one in v6
2. **Package Swap** — replace `react-router-dom` with `react-router@7`
3. **Import Rewrite** — update all imports from `react-router-dom` → `react-router`
4. **Deprecation Cleanup** — remove `json()`, `defer()`, handle new APIs
5. **Framework Mode** (optional) — adopt Vite plugin and file-based routing

## Future Flags (Phase 1)

| Flag | Risk | Key Change |
|------|------|------------|
| `v7_relativeSplatPath` | Medium | Relative paths in splat routes resolve from parent |
| `v7_startTransition` | Low | State updates wrapped in `React.startTransition` |
| `v7_fetcherPersist` | Low | Fetchers persist until unmount instead of returning to idle |
| `v7_normalizeFormMethod` | Low | `formMethod` is uppercase (`POST` not `post`) |
| `v7_partialHydration` | Medium | `<RouterProvider>` uses `initialData` instead of `fallbackElement` |
| `v7_skipActionErrorRevalidation` | Low | Loaders don't revalidate after 4xx/5xx action responses |

## Full Guide

Read [.github/docs/MIGRATION_GUIDE_v6_to_v7.md](.github/docs/MIGRATION_GUIDE_v6_to_v7.md) for the complete plan, automation scripts, verification checklists, and PR sequence.

## Agents

| Agent | Purpose |
|-------|---------|
| `@rr-migrate` | Execute migration steps |
| `@rr-verify` | Validate each step |
| `@rr-rca` | Investigate failures |
