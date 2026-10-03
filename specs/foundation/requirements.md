# Overy — Requirements

> Rules the code must obey: stack, MVP boundaries, schema invariants, conventions. Not “how it is built” (that is `specs/docs/product-architecture.md`) and not “what and when” (that is `specs/work/roadmap.md`).

## Stack & constraints

The canonical list of technologies. It is fixed: deviations need a decision.

| Layer     | What we use |
| --------- | -------------- |
| Runtime   | PHP **8.5**, Bun 1.3+, Node 24+ (LTS, for the Electron toolchain) |
| Backend   | Laravel 13, Symfony 8.x (resolved by composer) |
| Web server (prod) | Laravel Octane 2 — **FrankenPHP** long-running worker (`OCTANE_SERVER=frankenphp`; Forge-managed, nginx-proxy + supervisor). Dev/desktop: plain php-fpm/serve |
| Desktop   | NativePHP for Desktop 2.3 (Electron 40, Mac/Windows/Linux) |
| Mobile    | NativePHP for Mobile 3 Air (iOS/Android, Phase 7+) |
| Auth      | Fortify + Socialite (Google OAuth live; Apple/Facebook drivers code-complete, gated by `services.{p}.client_id` — held; Telegram HMAC, custom JS button) + Sanctum (per-device sync tokens + personal access tokens with ability `mcp`). Prod sign-in = Google + Telegram; email/password dev-only (the `EMAIL_AUTH` toggle opens public email signup) |
| Frontend  | Inertia v3 + Svelte 5 (runes) + Tailwind 4 + shadcn-svelte |
| Editor    | TipTap (ProseMirror) + tiptap-markdown (+ GFM tables, round-trip through markdown) |
| Admin     | Filament 5 + Livewire 4 (web-only, `/admin`) |
| Agent-surface | `laravel/mcp` 0.9 — `Mcp::web('/mcp/nodes')`, Sanctum ability `mcp` |
| Build     | Vite 8 (Rolldown) |
| DB cloud  | Postgres 18 (Contabo VPS, Phase 5) |
| DB device | SQLite + FTS5 (forced by NativePHP) |
| Storage   | Contabo Object Storage (S3-compatible): **all** attachment blobs on cloud (staging disk → `PushStagedBlobToS3Job`); desktop/Herd write to the local `vault` disk directly. There is no size threshold; there is a per-plan `StorageQuota` |

**Pinning notes:**

- **PHP 8.5.** The desktop bundle takes PHP 8.5 from upstream `nativephp/php-bin` 1.2.0+ (zips for mac/linux/win); there is no custom php-bin build. Details: `specs/docs/dev-commands.md`.
- **Symfony** is not pinned by hand; composer resolves it. `nativephp/desktop` 2.2.1 widened the constraint to `^6.4|^7.2|^8.0`, so Symfony 8.x now resolves freely.
- **Octane (prod web).** The driver is FrankenPHP, chosen at runtime through `OCTANE_SERVER` (the stock config default `roadrunner` is left alone; the binary/worker is gitignored, Forge installs it). Long-running worker invariant: **no request state in `singleton()`/`static`**. Per-request/per-owner memoizers are bound with `scoped()` (`AppServiceProvider`), and `Container::forgetScopedInstances()` clears them before every request. Details: `specs/docs/product-architecture.md` §1, skill `octane-development`.

## Architectural invariant: one Laravel app

**Overy = one Laravel app**, packaged into three runtime targets (web / desktop / mobile). A multi-app monorepo (`apps/cloud` + `apps/desktop` + `packages/core`) is **rejected** (rationale: `specs/work/observations.md`).

- Repo root = a standard Laravel skeleton, one `composer.json` (web + NativePHP deps).
- The runtime branches through env (`OVERY_DESKTOP`, `OVERY_FLAVOR`) and `App\Support\Environment`, not through separate codebases.
- The runtime picks the DB driver: NativePHP intercepts to SQLite in the bundle; web reads `.env` (Postgres locally and in prod).
- Cloud-only routes/controllers register under an `Environment::isWeb()` guard.
- `OVERY_FLAVOR=prod|dev` ⊥ `OVERY_DESKTOP` (prod/dev vs desktop/web).

## URL scheme

No subdomains. Web on the apex with a path prefix.

- **`/`** — the public marketing landing page (`inertia('landing')`, carries the `#pricing` section), web for guest and authed; a signed-in user enters the app through the “Open Overy” CTA → `/n`. Desktop: `/` redirects straight to `/n` (no marketing chrome). Login/registration: Fortify routes (`/login`, `/register`).
- **Public static pages** (root mount, guest+authed, any runtime): `/changelog` (`inertia('changelog')`, data-driven `lib/changelogReleases.ts`), `/legal` (`inertia('legal')`, single-page anchors `#terms/#privacy/#refund`, a Paddle MoR requirement), `/docs` + `/docs/{section}` (`DocsController`, markdown sources in `resource_path('docs')`, sequential prev/next). Shared `StaticPage` shell (`SiteHeader`/`SiteFooter`).
- **`/n/...`** — vault (`n` = nodes). Web: `Route::prefix('n')->middleware('auth')`. Desktop anonymous: the same prefix **without** `auth` (the Auth shim returns a fixed owner, so controller code does not tell the difference). Desktop linked: `auth` + Sanctum `device_token`. `/n/{key}` is a **thin resolving alias** (no redirect/backfill; stored tabs/cookies resolve): reserved view → Crockford id → owner slug.
- **Per-kind canonical list URLs** (root mount; each kind owns its slug namespace, which kills the cross-kind slug-collision class): `/views/{slug}`, `/states/{slug}`, `/types/{slug}`, classifier instances `/types/{grouper}/{instance}`, content node `/n/{id}`; each has a detail twin `…/{node}`. `/tags/{slug}(/{node})` is **legacy**, 301 → `/types/{grouper}/{slug}` (inline `#` chips know only the slug → they go through the 301 on purpose). Summary intersection drill-down: `/n/{key}/{dimKind}/{dimKey}(/{node})`. Resolver: `VaultKeyResolver::resolveKind` (kind-scoped, `VaultKind::ofNode`); `canonicalKey` is a wire-only `kind:key` token and does not reach the URL (the FE strips it to the bare key through Wayfinder). Reserved-slug write guards are removed (the namespace itself separates the `notes` state and the `notes` view).
- **`/settings/...`** — without `/n`. In Phase 9 the vault moves to client-side render; settings/billing stay server-side Inertia.
- **`/api/sync/...`**, `/api/auth/me` — sync API (Sanctum device token).
- **`/mcp/nodes`** — MCP node control plane (`routes/ai.php`, `NodeServer`), `auth:sanctum` + `CheckAbilities:mcp`. Tokens are minted in Settings → MCP & integrations. Every tool is scoped strictly to the acting user’s nodes. The Artisan CLI over the same operations was **removed on purpose**: there is one surface.
- Cloud endpoints: local Herd `http://overy.test` → staging `https://dev.overy.app` (branch `dev`) → prod `https://overy.app` (branch `main`; Phase 5 cutover done, interim `new.overy.app` archived). Each cloud environment has its own demo origin: `demo.dev.overy.app` / `demo.overy.app`. Environment invariants and what an environment needs besides the site: `specs/docs/dev-commands.md`.

## Schema invariants (cross-phase)

- **Cross-DB parity.** The same migrations on Postgres (cloud) and SQLite (device). `$table->json()` + `whereJsonContains`. FTS differs (FTS5 vs tsvector) → search abstraction in Phase 6.
- **FK constraints are not used** between domain tables. Integrity is an application-level cascade (`app/Services/Graph`, `app/Services/Sync`).
- **UUID primary keys (UUIDv7)** for everything that syncs: merge without collisions + time-ordered B-tree clustering. On SQLite `CHAR(36)` (BLOB(16) considered/rejected, low ROI). The FE-visible layer encodes UUID ↔ 26-char Crockford-Base32 (`IdPresentationEncoder`); the sync wire is UUID-native. Trade-off: the first 48 bits of v7 = creation ms-timestamp (acceptable for a personal vault; auth tokens stay v4).
- **Soft-delete** (`deleted_at`) for nodes; hard-delete tears down edges through `Cascade`.
- **Edge predicates: 5 live** (`child_of`=1, `extends`=2, `now_in`=4, `scoped_to`=6, `labeled_by`=7; numbers frozen). `labeled_by` is the single labeling axis (a merge of the retired `classified_by`=3 + `mentions`=5); the membership-vs-reference line is read-derived through `LabelKindResolver` from the nature of the target; `child_of` is **single-parent** (Cardinality::Single). Details: `specs/docs/product-architecture.md` §3.
- **Dual representation, one writer.** A link lives as an edge (SoT) and as a mirror in `parsed_meta`: scalars `parent_id`/`now_in`/`extends`, lists `classified_by[]`/`scoped_to[]`. All values are **UUIDs** (slugs collided between nodes with the same name). **Only** the edge-write seam `EdgeAttachmentService` writes the mirrors (`EdgePredicate::scalarMirrorKey()`/`listMirrorKey()` → `EdgeMirror`/`ListMirrorWriter`, through `StampedMetaWriter`: quiet save + per-field LWW stamp). The single exception is a manual plain label (not derived from an edge): `NodeEdgeApplicator` writes its subset, and the rebuild takes the union. Adding a second mirror writer is forbidden. On the wire `classified_by` travels **as a slug** (`ClassifiedBySlugWireFilter`): edge = SoT, slug = transport, uuid = local read cache. Details: `specs/docs/product-architecture.md` §3.
- **Node invariants** (`state ⇒ classifier`, classifier-side ⊥ instance-role traits — 5 traits {classifier,state,type,actionable,attachment}, acyclic edges, edge-level ortho: a classifier-side node ⊄ `now_in` state / `labeled_by → [state]` membership; a `scoped_to` source must carry `[classifier]`; state = mention only) — `NodeInvariantValidator` (write boundary) + edge `*Invariant` chain (`SelfEdge → TargetTrait → RequiredSourceTrait → SourceTrait → Cycle → LabeledByCycle`). Strict 422 on Create/Update/sync-apply. Full matrix: `specs/docs/node-invariants.md`.
- **Protected nodes.** Load-bearing built-ins (States grouper + Inbox/Next/Done/Cancelled) carry a durable identity flag `parsed_meta.protected`: non-deletable/non-archivable (`NodeProtection`, `StateSlug::isSystemCritical()`); sync replication is not gated.
- **Derived device-local columns are projections, not data.** `nodes.search_text` (plain text of the body) and `nodes.title_normalized` (`MentionLabel::normalize(title)`, index `(user_id, …)`) are derived from fields that already travel over the wire, so they are themselves **not** in the push allowlist or in `NodeHasher`; otherwise two machines would compute different hashes for one node. The cost: **every** writer must be covered, because a writer that forgets fails silently (Eloquent paths: the `saving` hook; a bulk insert that bypasses the model derives on its own; a new column: a forward migration with a backfill, including trashed rows if the contract reads them).
- **Crypto boundary.** All sync/controllers write through `ContentCodec` (DI): no direct `json_encode/decode` in domain code.
- **Shape gate before a query by `nodes.id`.** Values of the `parsed_meta` mirrors (`classified_by[]`, `now_in`, `extends`, `parent_id`, `scoped_to`) are not guaranteed to be uuids: the sync wire and frontmatter name classifiers by slug. Any such value that goes into a query by `nodes.id` must pass `Support\Identity\StoredId`; a slug in a uuid column on Postgres fails the whole query (`22P02`), and SQLite hides this silently. Resolving both forms: only `ClassifierLeafResolver`.
- **Migrations are forward-only.** Every schema change is a new migration file; existing migrations are not edited, and `migrate:fresh` is not applied to populated environments (a repeated `up()` on a populated cluster must no-op).

---

## Frontmatter & vocabulary

- **The frontmatter parser is the PHP single source of truth** (`app/Services/Frontmatter/Parser.php`). There is no duplicate TS parser; the FE gets the already parsed `parsed_meta`.
- **Built-in vocabulary: 26 nodes**, seeded by `database/seeders/MinimalTypeSeeder.php`: 7 type nodes (people / projects / meetings + groupers states / tags / contexts / areas, `sidebar:true`) + instances (7 states / 5 contexts / 4 areas / 3 tags `#idea`/`#link`/`#youtube`). **Display metadata (singular/plural/icon/instance_icon) lives in `App\Classifier\BuiltInClassifier`**; the seeder holds only templates + the `protected` flag and merges the two halves. The copy in the seeder was a hand-synced pair that a single parity test held together. Model/mechanics: `specs/docs/product-architecture.md`.
- **There is no built-in `tasks` classifier, and none is to be added.** `actionable` is an instance-role trait, not a type; the actionable surface is the state queues.

## Scope — what is **not** in the MVP

- ❌ Full per-line outliner + CRDT · promoting bullets without a checkbox · `checkbox_mode: log` · UI Trash (the soft-delete mechanics exist) · custom boards · Postgres FTS (PG currently has an ILIKE fallback, `Search.php`) · SSE realtime push — **Phase 6**. Differential auto-updates left this list: an open page stays fresh by pull (a heartbeat outside Inertia + `vaultRevision` on reconciling contracts), push stays deferred
- ❌ Folder mounting · agent loop — **Phase 8**. No CLI is built: the agent has one surface, MCP
- ✅ Shipped out of order (was Phase 6/8 scope, is in the code): pinned + Pinned-Next tab · suggestions/Spotlight (`/api/suggest`) · state board (grouped node-list) · Settings → devices/revoke · **MCP node control plane** (`/mcp/nodes`)
- ❌ `contact` / `agent` traits — **retired** (not MVP scope; 5 live traits)
- ❌ E2E encryption — **Phase 9**
- ❌ Inter-agent protocol — horizon
- ❌ iCloud/Dropbox/Syncthing/Git as fallback sync · Trait configs in a `_traits/` folder — **never**

## Code conventions — SOLID + GoF

Pragmatic, not ritual.

- **Single source of truth is the default, not a wish.** Any concept (order, state, classification, label) is stored in ONE place. A second mirror/cache/store of the same concept “for convenience/speed” is not added: that is exactly what breeds divergence bugs. Before adding a field/store/prop: **does this concept already live somewhere?** Yes → read it from there, do not add a second writer. Two signals of one concept on different surfaces → **collapse** them (retire the extra one) instead of adding a third read path. A different *scope* (per-context override vs global default) is legit, not a duplicate; two global orders of one set are a duplicate. Edge = SoT, scalar mirror = read-only cache, reconcile only through an explicit DTO field. A feature the owner has called “fragile in its sources of truth” is redesigned to SSoT at the plan stage, without a band-aid.

- **SRP / DI / OCP** are the default: one service = one job, constructor injection, new behaviors through listeners/observers, not if-branches.
- **Strategy** — `ContentCodec` (`Passthrough`/`Sodium`), `NodeMirror` (`Desktop`/`Null`), `VaultEventSource` (`ChildProcess`/`Null`).
- **Observer** — Laravel events (`VaultFileChanged`, `OpenedFromURL`, `MessageReceived`).
- **Adapter** — JS↔PHP bridges (`ChildProcessVaultEventSource`, NativePHP `EventWatcher`).
- **Command** — Artisan (`sync:tick`, `vault:scan`, `sync:link-cloud`).
- **When we do NOT introduce a pattern.** One driver = one class without an interface. A layer is added at the second concrete case; premature abstraction makes code worse.
- **Skills.** Architecture/refactor → activate `solid` + `gof-design-patterns`; domain specifics → project skills (see `CLAUDE.md` §Skills Activation).

### Vault read perf — one bulk funnel + bounded build (do not drift)

Every vault read surface (sidebar tree, counts, node-detail outliner, summary, list) **is derived from ONE** bulk snapshot `CategoryTreeAssembler::load()` (per-request singleton, memoized): no **per-node** queries in presenters. `load()` is the only place for graph fetches; off-walk entry goes through `firstLevelCountFor`/`outlineFor`/`presentInstanceRow` (priming wrappers). A per-node walk = N+1, which breaks create/edit on large nodes.

**Bounded build (perf wave 2026-07-28).** There is still one funnel, but the BUILD is limited to what the client actually looks at; “load everything, render everything” paid for what nobody sees:

- `categories(ownerId, $expandedIds)` builds **only the open branches** of the sidebar (`$expandedIds` comes from the expansion cookie, which has one writer; a closed row still carries `count`/`hasChildren`, so what is visible does not change). `null` = the legacy whole-tree contract for in-process callers and tests.
- `loadOutlineScope(ownerId, rootId)` loads **the subtree of the open node**, not the whole vault.
- Payload mass is a cost axis just like queries: props nobody renders are cut (classifier members of a node, member sidecar; External-labels source chips trimmed to a sample of five), not windowed.

**Three axes, each with its own budget test**: a flat query count does not mean fast.

| Axis | Test |
|-----|------|
| Query count, size-invariant (N+1 = “queries grow with data”); 20k tier under `STRESS_TESTS=1` | `tests/Feature/Vault/VaultReadBudgetTest.php` |
| Per-prop: queries, **duplicate SQL** (a byte-identical statement in one render) and **hydrated models** | `tests/Feature/Vault/VaultPropBudgetTest.php` |
| Counters | `tests/Feature/Vault/VaultCountsBudgetTest.php` |

- A new read surface or a new counter → a size-invariant budget test is **required**.
- The enforcer is these budget tests (they catch the symptom), not an arch rule: queries in `Presentation/` are legitimate (`load()` itself, O(1) scoped probes), so they cannot be banned wholesale.
- Profiling probes (read cost, by caller, PHP sampling): `scripts/perf-probes/`; measure on a Postgres vault, because SQLite and an un-ANALYZEd PG both lie.
