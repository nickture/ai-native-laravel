<laravel-boost-guidelines>
=== .ai/overy rules ===

# Overy — Project

> Why the product exists, which principles it follows and how knowledge about it is organized. Boost assembles this file and `specs/foundation/requirements.md` into `CLAUDE.md`, so every agent session has them.

## Mission

A local data substrate for a Human+AI supernode. `.md` files in the user’s folder → an agent works on top → a network of nodes joins through an open protocol.

A double usefulness test at every iteration:

- the product is useful **without AI**, otherwise nobody starts using it;
- the structure is a substrate for the **agent**, otherwise the next step is impossible.

## Principles

- **Local-first, files-first.** The source of truth is `.md` with YAML frontmatter in a chosen folder. SQLite is a derived index: delete it and rebuild. Mobile (Phase 7+) is SQLite-only (iOS sandbox). **Where exactly the principle applies:** on desktop, literally (`DesktopNodeMirror` writes files, chokidar reads them back). On cloud there are no files (`NullNodeMirror`), the SoT is Postgres, and export is the way out to files; the principle holds because the domain model stays expressible as files and any vault can be pulled out into a folder of `.md`.
- **No vendor lock-in.** VSCode / Obsidian / vim read the vault. Delete the app and the data stays.
- **Sub-100ms.** Desktop through a local Laravel in Electron: 5–20 ms round-trip. Web: 100–300 ms (and the user accepts that web is slower).
- **Schema-free evolution.** No required fields. A new use case → the user adds fields and a trait instead of asking for a feature.
- **Progressive disclosure.** The full schema is in code from day 1; the UI reveals it per node based on behavior, without a global “advanced mode”.
- **Crypto boundary from day 1.** All sync operations and controllers write through `ContentCodec`. MVP: `PassthroughCodec` (no-op, the server sees plaintext); Phase 9: `SodiumCodec` without rewriting domain code.
- **Design in code.** A new feature is built in the product right away: on its architecture, existing components and demo data, without a separate mockup. The first version may be rough; after that it is refined, not rebuilt from scratch. A new UI primitive appears only when no ready-made one exists, and the reason is recorded in `specs/foundation/design.md`.

## How project knowledge is organized

| Where | What lives there | How it reaches the agent |
|---|---|---|
| `specs/foundation/` | Why the product exists and by which rules: `project.md`, `requirements.md`, `design.md` (design system), `text.md` (readers, tone, vocabulary) | `project.md` and `requirements.md`: in every session through Boost. `design.md` and `text.md` load when the agent reads or edits the frontend and copy (symlinks in `.claude/rules/`) |
| `specs/docs/` | As built: architecture, UX, node invariants, commands | By link |
| `specs/work/` | What is in progress: `roadmap.md`, the debt registry `observations.md`, active plans, open phases | By link |
| `specs/archive/` | Completed plans and closed phases | Not read unless asked |
| Overy vault | Ideas, tasks and questions for the owner | Through MCP |

**Precedence.** Foundation outranks skill rules, `.ai/rules` and plans: where they disagree, Foundation wins. If the project deliberately does not follow a skill rule, the reason is recorded in Foundation. If `docs/` disagrees with Foundation, or the code disagrees with `docs/`, that is drift, and it is recorded in `specs/work/observations.md`.

**Lifecycle.** A new plan is written in `specs/work/plans/`. When the plan is done, its outcome is added to `docs/` or `foundation/`, and the file moves to `specs/archive/`. Links inside the archive are not fixed. A finding that is not fixed right away is recorded in `observations.md` in the same pass. A closed entry is deleted from there.

**Editing the core.** After editing `project.md` or `requirements.md`, run `php artisan boost:update`, otherwise `CLAUDE.md` and `AGENTS.md` keep the old copy.

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

=== foundation rules ===

# Laravel Boost Guidelines

## Foundational Context

This application is a Laravel application running on PHP 8.5. Always use the APIs that match the installed major version of each package — do not assume a version.

Before relying on a package's API, confirm its installed version:
- PHP packages: run `composer show --direct` to list direct dependencies with versions, or `composer show <vendor/package>` for a single package.
- JS packages: check `package.json` for the installed versions.

## Skills Activation

This project has domain-specific skills available in `**/skills/**`. You MUST activate the relevant skill whenever you work in that domain—don't wait until you're stuck.

## Conventions

- You must follow all existing code conventions used in this application. When creating or editing a file, check sibling files for the correct structure, approach, and naming.
- Use descriptive names for variables and methods. For example, `isRegisteredForDiscounts`, not `discount()`.
- Check for existing components to reuse before writing a new one.

## Verification Scripts

- Do not create verification scripts or tinker when tests cover that functionality and prove they work. Unit and feature tests are more important.

## Application Structure & Architecture

- Stick to existing directory structure; don't create new base folders without approval.
- Do not change the application's dependencies without approval.

## Frontend Bundling

- If a frontend change doesn't show in the UI or you get a "Unable to locate file in Vite manifest" error, run `bun run build` or ask the user to run `bun run dev` or `composer run dev`.

## Documentation Files

- You must only create documentation files if explicitly requested by the user.

=== boost rules ===

# Laravel Boost

## Tools

- Laravel Boost is an MCP server with tools designed specifically for this application. Prefer Boost tools over manual alternatives like shell commands or file reads.
- Use `database-query` to run read-only queries against the database instead of writing raw SQL in tinker.
- Use `database-schema` to inspect table structure before writing migrations or models.
- Use `get-absolute-url` to resolve the correct scheme, domain, and port for project URLs. Always use this before sharing a URL with the user.
- Use `browser-logs` to read browser logs, errors, and exceptions. Only recent logs are useful, ignore old entries.

## Searching Documentation (IMPORTANT)

- Use `search-docs` before changes that depend on Laravel ecosystem APIs, behavior, configuration, or version-specific syntax. Skip it for copy-only edits and other changes where package documentation is irrelevant. Reuse sufficient results already in context instead of searching again.
- Pass a `packages` array to scope results when you know which packages are relevant.
- Use multiple broad, topic-based queries: `['rate limiting', 'routing rate limiting', 'routing']`. Expect the most relevant results first.
- Do not add package names to queries because package info is already shared. Use `test resource table`, not `filament 4 test resource table`.

### Search Syntax

1. Use words for auto-stemmed AND logic: `rate limit` matches both "rate" AND "limit".
2. Use `"quoted phrases"` for exact position matching: `"infinite scroll"` requires adjacent words in order.
3. Combine words and phrases for mixed queries: `middleware "rate limit"`.
4. Use multiple queries for OR logic: `queries=["authentication", "middleware"]`.

## Project Rules

- This project contains committed, area-grouped rules in `.ai/rules` when that directory exists, including path-scoped framework guidelines under `.ai/rules/boost`. Before you enter plan mode or create/edit any file, you MUST first: open @.ai/rules/index.md (it maps file globs to rule files), read every rule file whose globs cover the path(s) in scope, and run `grep -rin 'keyword' .ai/rules` to catch what a path match alone misses. Do not write code until you have read and are following every matching rule. If `.ai/rules` does not exist, continue without it.

## Artisan

- Run Artisan commands directly via the command line (e.g., `php artisan route:list`). Use `php artisan list` to discover available commands and `php artisan [command] --help` to check parameters.
- Inspect routes with `php artisan route:list`. Filter with: `--method=GET`, `--name=users`, `--path=api`, `--except-vendor`, `--only-vendor`.
- Read configuration values using dot notation: `php artisan config:show app.name`, `php artisan config:show database.default`. Or read config files directly from the `config/` directory.

## Tinker

- Execute PHP in app context for debugging and testing code. Do not create models without user approval, prefer tests with factories instead. Prefer existing Artisan commands over custom tinker code.
- Always use single quotes to prevent shell expansion: `php artisan tinker --execute 'Your::code();'`
  - Double quotes for PHP strings inside: `php artisan tinker --execute 'User::where("active", true)->count();'`

=== php rules ===

# PHP

- Always use curly braces for control structures, even for single-line bodies.
- Use PHP 8 constructor property promotion: `public function __construct(public GitHub $github) { }`. Do not leave empty zero-parameter `__construct()` methods unless the constructor is private.
- Use explicit return type declarations and type hints for all method parameters: `function isAccessible(User $user, ?string $path = null): bool`
- Use TitleCase for Enum keys: `FavoritePerson`, `BestLake`, `Monthly`.
- Prefer PHPDoc blocks over inline comments. Only add inline comments for exceptionally complex logic.
- Use array shape type definitions in PHPDoc blocks.

=== deployments rules ===

# Deployment

- Laravel can be deployed using [Laravel Cloud](https://cloud.laravel.com/), which is the fastest way to deploy and scale production Laravel applications.

=== herd rules ===

# Laravel Herd

- The application is served by Laravel Herd at `https?://[kebab-case-project-dir].test`. Use the `get-absolute-url` tool to generate valid URLs. Never run commands to serve the site. It is always available.
- Use the `herd` CLI to manage services, PHP versions, and sites (e.g. `herd sites`, `herd services:start <service>`, `herd php:list`). Run `herd list` to discover all available commands.

=== tests rules ===

# Test Enforcement

- Add or update tests for behavior and logic changes when a test provides meaningful regression coverage.
- Pure copy, styling, and layout-only changes do not require new or updated tests.
- When test coverage applies, run the affected tests and ensure they pass.
- Test the changed behavior and its important failure modes, but do not add tests beyond them.
- Read the `testing-best-practices` skill before writing tests.

=== inertia-laravel/core rules ===

# Inertia

- Inertia creates fully client-side rendered SPAs without modern SPA complexity, leveraging existing server-side patterns.
- Components live in `resources/js/pages` (unless specified in `vite.config.js`). Use `Inertia::render()` for server-side routing instead of Blade views.
- ALWAYS use `search-docs` tool for version-specific Inertia documentation and updated code examples.
- IMPORTANT: Activate `inertia-svelte-development` when working with Inertia Svelte client-side patterns.

# Inertia v3

- Use all Inertia features from v1, v2, and v3. Check the documentation before making changes to ensure the correct approach.
- New v3 features: standalone HTTP requests (`useHttp` hook), optimistic updates with automatic rollback, layout props (`useLayoutProps` hook), instant visits, simplified SSR via `@inertiajs/vite` plugin, custom exception handling for error pages.
- Carried over from v2: deferred props, infinite scroll, merging props, polling, prefetching, once props, flash data.
- When using deferred props, add an empty state with a pulsing or animated skeleton.
- Axios has been removed. Use the built-in XHR client with interceptors, or install Axios separately if needed.
- `Inertia::lazy()` / `LazyProp` has been removed. Use `Inertia::optional()` instead.
- Prop types (`Inertia::optional()`, `Inertia::defer()`, `Inertia::merge()`) work inside nested arrays with dot-notation paths.
- SSR works automatically in Vite dev mode with `@inertiajs/vite` - no separate Node.js server needed during development.
- Event renames: `invalid` is now `httpException`, `exception` is now `networkError`.
- `router.cancel()` replaced by `router.cancelAll()`.
- The `future` configuration namespace has been removed - all v2 future options are now always enabled.

=== laravel/core rules ===

# Do Things the Laravel Way

- Use `php artisan make:` commands to create new files (i.e. migrations, controllers, models, etc.). You can list available Artisan commands using `php artisan list` and check their parameters with `php artisan [command] --help`.
- If you're creating a generic PHP class, use `php artisan make:class`.
- Pass `--no-interaction` to all Artisan commands to ensure they work without user input. You should also pass the correct `--options` to ensure correct behavior.

### Model Creation

- When creating new models, create useful factories and seeders for them too. Ask the user if they need any other things, using `php artisan make:model --help` to check the available options.

## APIs & Eloquent Resources

- For APIs, default to using Eloquent API Resources and API versioning unless existing API routes do not, then you should follow existing application convention.

## URL Generation

- When generating links to other pages, prefer named routes and the `route()` function.

## Testing

- When creating models for tests, use the factories for the models. Check if the factory has custom states that can be used before manually setting up the model.
- Faker: Use methods such as `$this->faker->word()` or `fake()->randomDigit()`. Follow existing conventions whether to use `$this->faker` or `fake()`.
- When creating tests, make use of `php artisan make:test [options] {name}` to create a feature test, and pass `--unit` to create a unit test. Most tests should be feature tests.

=== laravel-octane/core rules ===

# Laravel Octane

This application uses Laravel Octane, a long-running PHP server. The application bootstraps once and handles many requests within the same process.

- Never store request-specific state in singletons or static properties, because it can leak across requests.
- Use `config('octane.server')` to detect the active driver (`swoole`, `roadrunner`, or `frankenphp`).
- Prefer scoped bindings (`$this->app->scoped()`) over singletons for per-request services.

When working on Octane-specific features (concurrency, shared tables, memory, driver configuration, testing), invoke `octane-development` for detailed rules.

=== wayfinder/core rules ===

# Laravel Wayfinder

Use Wayfinder to generate TypeScript functions for Laravel routes. Import from `@/actions/` (controllers) or `@/routes/` (named routes).

=== pint/core rules ===

# Laravel Pint Code Formatter

- If you have modified any PHP files, you must run `vendor/bin/pint --dirty --format agent` before finalizing changes to ensure your code matches the project's expected style.
- Do not run `vendor/bin/pint --test --format agent`, simply run `vendor/bin/pint --format agent` to fix any formatting issues.

=== pest/core rules ===

# Pest

- This project uses Pest. Create tests with `php artisan make:test --pest {name}`.
- Do not include the test suite directory in `{name}`. Use `SomeFeatureTest`, not `Feature/SomeFeatureTest`.
- Read the `testing-best-practices` skill for guidance on coverage, naming, structure, dependency isolation, and review.
- Do not delete tests or test files without approval. They are part of the application.

## Running Tests

- Run the narrowest set of tests that covers the change. Pass a file path or `--filter=testName` to `php artisan test --compact`.
- Rerun a test after each change to it.
- Run `vendor/bin/pest` to call the test runner directly. It accepts the same file path and `--filter=testName` arguments.
- After the feature tests pass, ask the user to run the complete suite with `php artisan test --compact`.

=== inertia-svelte/core rules ===

# Inertia + Svelte

- IMPORTANT: Activate `inertia-svelte-development` when working with Inertia Svelte client-side patterns.

</laravel-boost-guidelines>
