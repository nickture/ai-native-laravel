# Graph View (`/n/graph`)

## Context

Vault needs an Obsidian/Anytype-style whole-vault graph: first item in the sidebar Special Views, hotkey `G`, full-area graph in place of node-list + details (one merged cell with the card styling). Visual vocabulary = landing2 hero graph (shapes by kind, 3 edge styles by predicate, hover-dim). User decisions (final, after iterations):

- **Render: Canvas 2D for everything** (final, after iterations). `NodeGraph.svelte` / landing2 are the **visual reference only** — the canvas renderer reproduces their vocabulary; NodeGraph is NOT mounted. The shareable, DOM-free parts of that vocabulary (types, shape geometry params, `wavyPath()` math, `REL_LABEL`, edge style constants) are extracted into a pure-ts module the canvas consumes now and `NodeGraph.svelte` can adopt later (after its in-flight rework lands). Physics = **d3-force** (new deps `d3-force` + `@types/d3-force` — approved; API verified via context7: per-link distance accessors, variable-radius `forceCollide`, `fx/fy` pinning, `alphaTarget` reheat, manual `tick(n)` sync settle) — the library buys correct force layout AND perf headroom.
- **Predicate trio** child_of/classified_by/mentions shown; now_in/extends hidden. **ALL orphans visible** — nodes without edges just float (no vocabulary-orphan filter).
- **Node size scales with degree** (more edges = bigger). Zoom/pan mandatory.
- **Click → native tab with dedupe**; hover richer than Obsidian (see D4).
- Current vault scale: 1049 live nodes / 906 trio edges (local dogfood DB) — static-after-settle DOM is fine; canvas is the documented fallback trigger if it ever chugs (YAGNI now).

**Parallel-session coordination:** `resources/js/components/graph/NodeGraph.svelte` (extracted hero graph), `resources/js/pages/landing2.svelte`, `resources/css/app.css` are mid-rework (uncommitted user changes). DO NOT touch them. The shared glyph/styles are built as NEW files; pointing NodeGraph at them is a follow-up after the user's rework lands.

On execution start: mirror this plan to `specs/plans/graph-view-execution.md` (repo convention).

## Verified integration facts

- `SpecialViewFilter::apply()` `match` is exhaustive over `VirtualView` (app/Services/Vault/Filters/SpecialViewFilter.php:58) — new case **requires** an arm: `VirtualView::Graph => $query->whereRaw('1 = 0')` (graph never lists rows).
- `IdPresentationEncoder` rewrites only whitelisted keys (`id`, `from_id`, `to_id`, …) — wire edges as `{from_id, to_id, predicate}`, NOT `{a,b}` (would leak raw UUIDs). FE maps to `{a,b,rel}`.
- `NavItem` hides badge at `count === 0`; `VaultCounts`, `UserPreferencesController` (`Rule::enum(VirtualView)`), `TaxonomyPresenter`, `tabMeta.ts` pick the new case up automatically.
- Compile/test-forced mirrors: `resources/js/lib/vault/virtualViews.ts` (pinned by `TaxonomyTsDriftTest`) + `FLUENT_VIEW_ICONS` in `resources/js/lib/fluentIcon.ts` (`Record<VirtualViewSlug,…>`).
- `LoadVaultPage::props()` closures all evaluate on full visits → graph payload must NOT live there. `resolveSelected()` falls back to latest-node → controller must override `node => null` on graph.
- Tab dedupe blocks exist: `tabsStore.tabs[].key`, `activate(id)`, `addBlank()`; visit pattern = Spotlight.svelte:29-39.
- `@iconify/svelte` exports `loadIcon()` — icon rasterization without new deps. `simulation.find(x,y,r)` does hit-testing — no d3-zoom/d3-quadtree.
- Tests run SQLite, prod web Postgres → payload queries portable Eloquent only; orphan-vocabulary filter in PHP via `NodeTraits::of()`.
- Titles may carry `[[mention]]` tokens → canvas labels run `flattenMentions()` (lib/mentions/parseTitle).

## Design decisions

**D1 — graph prop loading.** Conditional closure in `VaultController` (precedent: `listContext`):
```php
if ($listRoute->internalView === 'graph') {
    $props['graph'] = fn (): array => IdPresentationEncoder::encode($this->graphPresenter->payload($ownerId));
    $props['node'] = fn () => null; // skip resolveSelected latest-node fallback
}
```
Direct visit ships payload in first response. In-app nav uses `only:[...SIDEBAR_NAV_PROPS]` → `graph` absent → `GraphView` fires `router.reload({ only: ['graph'] })` on mount, pulsing skeleton meanwhile; stale payload renders immediately + refreshes in background.

**D2 — wire shape** (raw UUID from presenter, Crockford via encoder):
```
graph: {
  nodes: [{ id, title, canonicalKey, icon, color, traits, ownSlug }],  // glyph = NodeGlyphData::toCosmetics()
  edges: [{ from_id, to_id, predicate }]                               // predicate = EdgePredicate slug
}
```

**D3 — payload query plan** (≤3 queries, pinned by `expectsDatabaseQueryCount`):
1. Live nodes: `whereNull('deleted_at')->whereNull('archived_at')->get(['id','user_id','title','parsed_meta'])`.
2. Edges: `whereIn('predicate', [ChildOf, ClassifiedBy, Mentions])` + `whereIn('from_id'/'to_id', liveSub)` — edges to dead endpoints drop by construction.
3. PHP: NO orphan filtering — every live non-archived node ships, edge-less nodes float free (user decision). `canonicalKeyFor()` per node (singleton resolver, one GROUP BY).

**D4 — tuning constants** (one `lib/vault/graph/constants.ts`):
- **Degree-scaled node radius (user requirement): `r = clamp(8 + 3·√degree, 8, 30)`** world units; `forceCollide(n => r(n) + 4)`; label font scales mildly with r. Shape+color still carry kind (landing2 vocabulary), size now carries weight.
- forceLink distance: child_of 60 / classified_by 90 / mentions 110; strength .5 / .2 / .2. `forceManyBody(-180, distanceMax 450)`, `forceCenter` + weak `forceX/Y(0.02)`.
- Zoom clamp `k ∈ [0.15, 4]`, wheel-zoom about cursor, ctrl+wheel = pinch, drag empty = pan, drag node = pin + reheat (`alphaTarget(0.3)`).
- Labels: `alpha = clamp((k − .6)/.3, 0, 1)`. Icons drawn when `k ≥ .45` and raster cached.
- **Hover (user-refined, richer than Obsidian)** via `simulation.find(wx,wy,24/k)`:
  - Node hover: hovered node + ALL incident edges + neighbor nodes at full opacity; rest dimmed to .14. Labels of hovered + all neighbors forced visible regardless of zoom (Obsidian parity). Incident edges keep their per-predicate styling (solid/dashed/wavy + color) at full opacity + 1.5px — the edge KIND stays readable, which Obsidian's uniform lines lack.
  - Info chip (DOM overlay, not canvas): landing2 `node-chip` recipe near the hovered node — title + kind/trait chips + connection count (+ state for actionables). Crisp text, theme tokens, no canvas text scaling issues.
  - Edge hover (fat invisible hit path, landing2 `.graph-hit` analog → canvas distance-to-segment test): edge lit + midpoint predicate label chip (`contains` / `tagged` / `mentions`), endpoints + their labels lit.

**D5 — theme** (`graphTheme.ts`, re-resolved on `documentElement` class MutationObserver):
child_of = `--muted-foreground` @.32 solid; classified_by = #9a7bd0 (dark #a78bfa) @.4 dash [5,4]; mentions = #5b86d6 (dark #60a5fa) @.36 wavy (canvas port of landing2 `wavyPath()` :347); node fill #fff (dark `--color-neutral-800`); icon/label colors = computed color of `COLOR_CATALOG` className via hidden DOM probe, cached per theme.

**D6 — position persistence**: module-level `Map<nodeId,{x,y}>` (`positionCache.ts`) — warm-start `alpha(0.05)` on re-entry; no server persistence (YAGNI).

**D7 — reduced motion**: `prefers-reduced-motion` → `simulation.tick(300)` sync, static render; reheats brief (`alphaTarget(0.1)`, raised decay).

## Milestones

Gate per milestone: `vendor/bin/pint --dirty --format agent` → `vendor/bin/rector process` → `vendor/bin/phpstan analyse` → targeted Pest; FE also `bun run types:check` + `bun run test`; **no `bun run build`** (HMR dev running). Commit per milestone. All `.svelte` edits via **svelte-file-editor agent + svelte MCP autofixer** (svelte 5 + inertia-svelte + tailwindcss skills).

### M1 — Reserve the view (backend TDD)
1. Red — `tests/Feature/Vault/GraphViewTest.php` (model: AttachmentsViewTest): `isReserved('graph')`; `label() === 'Graph'`; resolver → virtual route, internalView `graph`, reserved wins over user slug `graph` (node key degrades to Crockford); `LoadVaultPage::handle(…,'graph',…)['nodes']` empty; slug write `graph` rejected (`NotReservedSlug`).
2. Green — `case Graph = 'graph';` in `app/Enums/VirtualView.php` + label arm; `whereRaw('1 = 0')` arm in `SpecialViewFilter`.
3. `TaxonomyTsDriftTest` red → mirror in `virtualViews.ts`; add `graph: 'fluent:molecule-20-regular'` to `FLUENT_VIEW_ICONS`.
4. Commit `feat(graph): reserve the graph virtual view`.

### M2 — GraphViewPresenter + controller (backend TDD)
1. Red — `GraphViewPresenterTest`: empty vault; trio edges surface, now_in/extends excluded; deleted/archived nodes + their edges dropped; edge to deleted target dropped; orphans (vocabulary AND content) included as edge-less nodes; cross-owner isolation; rows carry `canonicalKey` + exact `toCosmetics()` fields; query-count pin.
2. Green — `app/Services/Vault/Presentation/GraphViewPresenter.php` (`final readonly`, ctor `VaultKeyResolver`; sibling model: `MentionIndexPresenter`). Reuse `NodeTraits::of()`, `NodeGlyphData`, `EdgePredicate->slug()`.
3. Red — `GraphPageTest` (HTTP): `GET /n/graph` → Vault page, `mode virtual`, `key graph`, `node null`, `nodes []`, graph payload with Crockford ids; partial `only:['graph']` works; SIDEBAR_NAV_PROPS partial omits graph; `/n/inbox` has no `graph` key.
4. Green — D1 wiring in `VaultController`.
5. Commit `feat(graph): graph payload presenter + /n/graph page props`.

### M3 — Shell + navigation (frontend)
1. `bun add d3-force && bun add -d @types/d3-force`.
2. `VaultShell.svelte`: optional `full?: Snippet` → one `col-span-2` cell with `pb-2 pr-2` wrapper + `rounded-xl bg-neutral-150 dark:bg-neutral-900` inner, skip list/card cells + Seam 2; Seam 1 stays.
3. `Vault.svelte`: `isGraph = key === VIRTUAL_VIEWS.Graph && mode === 'virtual'` → render `full` snippet with `GraphView`; tab-sync `$effect` already covers it.
4. `VaultSidebar.svelte`: prepend `'graph'` to `VIEW_ORDER`; hoist-to-front when saved order predates graph (mergeSavedOrder appends new keys at bottom — splice to index 0, comment the product decision); kbd map `VIEW_KBD: Partial<Record<VirtualViewSlug,string>> = { recent: 'R', graph: 'G' }` replacing the inline ternary.
5. `VaultHotkeysMount.svelte`: `case 'g'/'G' → visitView('graph')` + doc comment.
6. Placeholder `GraphView.svelte` (skeleton + D1 mount-reload `$effect`) — route fully navigable pre-canvas.
7. Gate + manual smoke `http://overy2.test/n/graph`. Commit `feat(graph): graph view shell, sidebar entry, G hotkey`.

### M4 — Canvas engine (frontend, vitest-first on pure modules)
`resources/js/lib/vault/graph/` (each `.ts` + colocated `.test.ts`):
- `vocabulary.ts` — the shared visual-reference module (NEW, DOM-free): `NodeShape`/`EdgeRel` types, shape geometry params, `wavyPath()` math, `REL_LABEL`, edge style constants — ported from `NodeGraph.svelte` (reference only; that file stays untouched, may adopt this module after the user's landing rework lands);
- `types.ts`; `graphData.ts` (wire→sim mapping, degree computation, `flattenMentions`, `shapeFor`: contact→triangle, actionable|attachment→square, project-classified→diamond, else circle; icon/color via `nodeVisuals` — never re-derived);
- `simulation.ts` (D4 forces incl. degree-radius + collide; warm-start; reduced-motion settle; `tick/reheat/pinNode/releaseNode/find`);
- `camera.ts` (zoomAt/clamp/screenToWorld/DPR); `labels.ts` (fade math); `graphTheme.ts` (D5); `iconRaster.ts` (loadIcon → offscreen atlas keyed `icon|color|dpr|theme`); `renderer.ts` (humble draw loop: edges solid/dashed/wavy → shapes (radius by degree) → icons → labels → hover-dim; rAF only while `alpha > min` or interacting); `positionCache.ts` (D6).
- `components/vault/graph/GraphView.svelte`: canvas bound to cell size + DPR; pointer/wheel → camera/drag/hover; click → tab dedupe:
```ts
const existing = tabsStore.tabs.find((t) => t.key === node.canonicalKey);
if (existing) tabsStore.activate(existing.id);
else tabsStore.addBlank();
router.visit(vault(node.canonicalKey).url, { preserveScroll: true, preserveState: true, only: [...SIDEBAR_NAV_PROPS] });
```
Commit `feat(graph): canvas force-graph renderer`.

### M5 — Polish + verification
Empty-vault state (mirror Vault.svelte empty card), cursors, canvas `aria-label`, force tuning, full verification. Commit `refine(graph): empty state, tuning, a11y`.

## Corner cases (each = test, per plan-contract rule)

Empty vault; orphans (vocabulary + content) included edge-less; deleted/archived filtered; edge-to-deleted dropped; now_in/extends never ship; reserved-slug collision `graph`; Crockford on wire (id/from_id/to_id); `/n/inbox` carries no graph prop; nodes [] + node null on graph; tab dedupe (vitest on extracted helper); G ignored while typing/overlay (shared guards); dark⇄light live re-render (graphTheme test); SQLite⇄Postgres parity (portable scopes); stale saved sidebar order → hoist; degree-radius math (vitest); reduced motion (manual).

## Verification

1. `php artisan test --compact --parallel` full suite + coverage ≥80% on new files (`composer run test:coverage`).
2. `vendor/bin/phpstan analyse` (level 10, 0 errors), `vendor/bin/rector process`, `vendor/bin/pint --dirty --format agent`.
3. `bun run test`, `bun run types:check`; svelte MCP autofixer over new/edited `.svelte`.
4. Manual on Herd `http://overy2.test/n/graph`: Graph first in sidebar, kbd G, no badge; merged cell matches card styling; shapes/edge styles = landing2; **hub nodes visibly larger (degree scaling)**; **hover: neighborhood lit + neighbor labels forced + info chip + per-predicate edge styling readable; edge hover shows predicate label**; wheel/pinch zoom + pan + node drag; labels/icons fade by zoom; node click opens/activates correct tab, graph tab stays; theme flip live; archive node → reload → gone; reduce-motion static.

## Design review (SOLID/GRASP/GoF/KISS/DRY/YAGNI)

- **SRP/Information Expert**: payload in new `GraphViewPresenter` (sibling: `MentionIndexPresenter`), not `LoadVaultPage` (every-view coordinator; graph data is view-conditional + heavy).
- **OCP/Strategy**: rides `VirtualView` enum + `ViewFilter` registry; drift test + `Record<VirtualViewSlug,…>` make forgotten consumers compile/test failures.
- **DRY/Flyweight**: `VaultKeyResolver` singleton, `NodeGlyphData`, FE `nodeVisuals` + `COLOR_CATALOG` — zero re-derivation. `vocabulary.ts` = single source of the graph's visual language for canvas now, `NodeGraph.svelte` later (no coupling to the in-flight landing rework).
- **Humble Object**: sim/camera/labels/theme = pure vitest-able `.ts`; svelte + renderer thin over canvas.
- **Proxy/cache**: icon raster atlas, theme token cache (observer-invalidated).
- **KISS**: no d3-zoom/quadtree (manual camera + `simulation.find`); conditional closure over new Inertia mechanisms; PHP-memory orphan filter over cross-driver JSON SQL.
- **YAGNI**: no server layout persistence, no pagination, no live diffing.
