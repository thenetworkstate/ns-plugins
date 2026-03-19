---
name: ns-internal-dashboard
description: Conventions, patterns, and gotchas for the NS admin dashboard. Use this skill when working on any code under the (admin) route group, admin tables, admin analytics charts, admin filters, impersonation, or the analytics batcher. Trigger whenever the user mentions admin dashboard, internal dashboard, admin tables, admin users table, leases table, unit table, admin analytics, admin charts, KPI cards, impersonation, or any feature under /admin routes — even if they don't say "dashboard" explicitly.
---

# NS Internal Dashboard

The admin dashboard lives at `apps/web/app/(admin)/admin/`. It's a Next.js 15 app with server-side Privy auth, client-side admin gating, and a priority-based analytics query batcher.

## Architecture

```
apps/web/app/(admin)/
├── layout.tsx                    # Auth enforcement (Privy) — all child pages inherit this
├── AdminNav.tsx                  # Top navbar with impersonation status
└── admin/
    ├── _components/shared/       # Shared UI: charts, date utils, sidebar, tooltips
    ├── _lib/                     # Core utils: analyticsBatcher, formatters, types
    ├── users/                    # Users table feature
    ├── unit-lease-table/         # Leases table feature
    ├── bookings/                 # Booking management
    ├── payments/                 # Payment management
    └── [other-features]/
```

## Auth

Auth is enforced at the layout level via `getPrivyTokenOrRedirect()`. Individual pages don't need auth checks — if the layout renders, auth has passed. The `AdminWrapper` component gates access to core team members on the client side.

After any auth context change (like starting/ending impersonation), clear sessionStorage and call `router.refresh()`:

```typescript
window.sessionStorage.removeItem('ns:initViewer');
router.refresh();
```

Forgetting `router.refresh()` after impersonation changes leaves stale auth context, which causes confusing UI bugs.

## Analytics Batcher

`_lib/analyticsBatcher.ts` prevents database overload by batching and prioritizing queries. This is the most important system to understand because naming a query wrong silently degrades page load performance.

### How it works

All `scheduleBatchQuery()` calls within a render cycle are collected, then flushed on the next macro tick. Queries are grouped by priority, and each priority level waits for the previous one to finish. Within a level, queries fire in parallel batches of 4.

### Priority groups

| Priority | Group | Name pattern | What it powers |
|----------|-------|-------------|----------------|
| 0 | `kpi` | `main_*` | KPI cards at top of page |
| 1 | `traction` | `acquisition_full_funnel`, `retention_population_time_series_summary` | Section right below KPIs |
| 2 | `default` | Everything else | Chart summaries |
| 3 | `slow` | `retention_cohort_*`, `revenue_*` | Heavy analytical queries |
| 4 | `detail` | `*_detail` | Large row sets, drilldowns |

Auto-detection uses name prefixes. If a query name doesn't match the convention, it falls to `default` (priority 2). Override with an explicit group parameter when auto-detection gets it wrong:

```typescript
// Auto-detected as kpi (priority 0) because it starts with main_
scheduleBatchQuery<KpiData>('main_kpi_summary', { month: currentMonth });

// Auto-detected as detail (priority 4) because name contains _detail
scheduleBatchQuery<DetailData>('users_detail', { filters });

// Override: this is a heavy query but doesn't match the slow prefix
scheduleBatchQuery<CustomData>('custom_heavy_report', { params }, false, 'slow');
```

**Single query goes to** `/api/v1/internal/analytics/run/`. **Multiple queries go to** `/api/v1/internal/analytics/batch/`. Identical queries are deduplicated — all subscribers get the same result.

## Table Features

Each table feature follows a consistent file structure. Follow this when creating new tables:

```
admin/[feature]/
├── page.tsx
├── _components/
│   ├── [Feature]Table.tsx                    # Main table component
│   ├── [feature]-table-name.columns.tsx      # TanStack column definitions
│   ├── [feature]-table-name.formatters.tsx   # Cell value formatters
│   ├── [feature]-table-name.constants.ts     # Filter options, column options, storage keys
│   ├── [feature]-table-name.presets.ts       # Saved filter/column presets
│   ├── [feature]-table-name.hooks.ts         # Table-specific hooks
│   └── [feature]-table-name.utils.ts         # Filter builders, comparators
```

### Filter state

Filter state is a `Record<FilterKey, string | null>` where every key is always present. Missing values are `null`, never `undefined`. Always construct filter state through `buildFilterState()`:

```typescript
// Correct — normalizes all keys with null defaults
const filters = buildFilterState({ user_type: 'member', on_campus: 'this_month' });

// Wrong — missing keys will cause comparison bugs
const filters = { user_type: 'member', on_campus: 'this_month' };
```

The reason this matters: `areFiltersEqual()` compares every key in the filter state. If a key is missing, the comparison breaks and presets stop auto-detecting.

### Presets

Presets are saved filter + column visibility + ordering combinations. They sync to the URL via `?preset_id=` query param. When filters change, `getActiveAdminUsersTablePresetId()` checks if the active filters match any preset and highlights it in the sidebar.

Preset filters are functions `(now: dayjs.Dayjs) => FilterState` so they can compute time-dependent defaults.

### Column visibility

Stored in localStorage (key per table, e.g. `admin-users-visible-columns`). Read happens on component mount — localStorage changes don't reflect until next page visit. Use React state for immediate UI updates, then persist to localStorage as a side effect.

### Virtual scrolling

Tables use virtual scrolling with a configurable prefetch distance:

```typescript
const SCROLL_PREFETCH_ROWS = 10;
const ROW_HEIGHT_PX = 48;
export const SCROLL_PREFETCH_PX = SCROLL_PREFETCH_ROWS * ROW_HEIGHT_PX; // 480px
```

## Formatters

All formatters in `_lib/formatters.tsx` and feature-specific formatter files follow these rules:

1. Handle null/undefined inputs — return `renderEmptyValue()` which renders a styled dash `—`
2. Be pure functions — no side effects, no hooks, no fetch calls
3. Normalize before comparing — user types come in varied formats (`long-term`, `longterm`, `long_term`)

```typescript
// Use isLongtermUser() for boolean checks — never compare raw strings
if (isLongtermUser(userType)) { ... }

// formatUserType() handles all normalization
formatUserType('long-term')  // → 'Longterm'
formatUserType('longterm')   // → 'Longterm'
formatUserType(null)         // → <styled dash>
```

## Date Handling

Centralized in `_components/shared/date.ts`. Use these functions instead of manual date formatting.

- **Month keys**: `YYYY-MM` format. Use `formatMonthLabel()` → `'Jan 25'`, `formatMonthLabelFull()` → `'Jan 2025'`
- **Week keys**: `YYYY-MM-WN` format (N = 1–4). Use `formatWeekLabel()`, `formatWeekLabelRange()`
- **Projection multiplier**: `getProjectionMultiplier()` extrapolates partial-month data. Only works on current month — always check `isCurrentMonth()` first

## Data Fetching

- **Client components**: Use `nsFetch` — it auto-detects environment
- **Server Components / server actions**: Use `serverFetch` with explicit token forwarding
- Mixing them causes hydration mismatches

## React Query Keys

Follow the hierarchical factory pattern for selective invalidation:

```typescript
const root = ['admin', 'user', userId];
const profile = [...root, 'profile'];
const payments = [...root, 'payments'];

// Invalidate just payments
queryClient.invalidateQueries({ queryKey: payments });

// Invalidate entire user subtree
queryClient.invalidateQueries({ queryKey: root });
```

After impersonation changes, invalidate both:
```typescript
queryClient.invalidateQueries({ queryKey: ['admin', 'impersonation', 'status'] });
queryClient.invalidateQueries({ queryKey: ['user', 'profile'] });
```

## Telemetry

All admin events go to PostHog with the `admin` prefix and `admin_telemetry` feature group. Auth-related events use the `admin_auth` prefix. New admin telemetry events should follow the same convention.

## Metadata

Page titles are generated from the route pathname in `layout.tsx` via `getAdminPageName()`. When adding a new admin page, add a mapping there:

```typescript
if (normalizedPath.startsWith('/your-new-page')) return 'Your Page Name';
```

## Quick Reference

| Pattern | Where | Key |
|---------|-------|-----|
| Filter state | `buildFilterState()` in feature utils | All keys present, null for missing |
| Column prefs | localStorage | Read on mount only |
| Query priority | `scheduleBatchQuery()` group param | Name prefix auto-detects |
| Dates | `_components/shared/date.ts` | Never format manually |
| Formatters | Feature `formatters.tsx` | Handle null, return `renderEmptyValue()` |
| Auth | Layout-level | No per-page checks needed |
| Impersonation | `_lib/impersonationStatus.ts` | Clear session + refresh after changes |
| Telemetry | PostHog | `admin_` prefix, `admin_telemetry` group |
