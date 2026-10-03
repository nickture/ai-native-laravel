# Grouped node-list → board (all headers visible, per-group lazy load)

## Context

Node/classifier/type views render children grouped by state. Current bug
(post seamless-scroll work): a group's header shows its true server count
(`groupCounts`/M2) up-front, but its ROWS are paginated far down the
contiguous stream (M1 contiguity ranks custom/scoped states to the orphan
tail, behind inbox + the big terminal `done`). So a non-collapsed group
renders an empty expanded header; toggling it `router.reload(reset)` resets
to page 1, which never reaches the tail rows → "expand does nothing".

User decision: make it a **board** — every group header always visible;
clicking a collapsed group lazily loads **only that group**; the toggled
header stays anchored (no scroll jump). Collapsed groups are not loaded.

## Design

Grouped views are bounded (one container's membership, ≤2000). Drop
infinite-scroll pagination FOR GROUPED VIEWS; keep it for flat unbounded
views (recent/all/today/…).

- **Initial load** (1 request): grouped view returns ALL **non-collapsed**
  group rows in one page (collapsed excluded → not loaded). Default-visible
  groups show their rows immediately; no jerk (hasNext=false).
- **Board headers**: all groups with a server count render (existing
  `groupNodesByState` keep-rule + `groupCounts`).
- **Per-group lazy** (1 request each, on demand): expanding a collapsed
  group fetches ONLY that group's rows and appends to a local buffer.
  Collapsing hides + caches (re-expand instant). Header anchored.

## Milestones (atomic commit + TDD each)

### B2 — whole-load non-collapsed for grouped views (server; ship first)
Fixes the visible bug immediately. `VaultListQuery::groupsByView()`;
`LoadVaultPage` nodes closure: grouped → `loadList(page:null, collapsed)`
wrapped in a 1-page paginator (hasNext=false); flat → unchanged window.
Tests (Pest): grouped `nodes` returns all non-collapsed (no 50-cap);
collapsed still excluded; flat view still paginated (50-cap honoured).

### B1 — per-group lazy fetch (server + FE)
`GET /api/views/{view}/group/{slug}` (`VaultGroupController`) → that group's
rows (reuse `VaultListQuery`). FE: local `extraGroupRows` buffer merged via
`uniqueById`; expand collapsed group → `useHttp` fetch → append; collapse →
hide + persist cookie (no fetch); re-expand cached → instant; anchored
header. Replace `onToggleGroup`'s whole-reload with per-group append.
Tests: endpoint scopes to view+slug (children only, actionable+now_in,
stateless sentinel), count == `groupCounts[slug]`; vitest buffer/append.

### B3 — board polish (FE)
Verify board keep-rules (every count>0 header renders; empty global hidden,
scoped shown); drop dead `expandNeedsFetch`/reset path; cleanup. vitest.

## Design review (SOLID/GoF/GRASP/KISS/DRY)
- **SRP/Information Expert**: whole-vs-window decision in `LoadVaultPage`
  (owns prop shaping); group filter in `VaultListQuery` (owns the ordered
  set); per-group endpoint reuses `loadList` (no duplicated query logic).
- **OCP**: `groupsByView` reuses the `ViewFilter` Strategy — new view
  families stay open/closed. Per-group fetch is additive (new route +
  local buffer), existing flat path untouched.
- **DRY**: B1 endpoint reuses `loadList` + the same row encoder as the
  list; FE reuses `uniqueById` + `groupNodesByState` + `pinAnchor`.
- **KISS/YAGNI**: no per-group pagination (groups bounded) — `// ponytail`
  note + upgrade path if a single non-collapsed group ever gets huge.
  No new abstraction for the buffer (plain `$state` array).

## Verification
Per milestone: changed-file Pest (`--parallel --compact`); after each,
Pint + Rector + PHPStan L10 (0 baseline) + vitest + svelte-check. Dogfood
on Herd: open the Overy project — filament/bugs/etc show rows; collapse a
group → header+count stays, rows hidden; expand → its rows load, header
stays put.
