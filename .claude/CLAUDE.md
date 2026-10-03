# Overy — project guidelines

Project context for Claude Code on top of the root `CLAUDE.md`. Boost assembles the Foundation core, `specs/foundation/project.md` and `requirements.md`, into the root `CLAUDE.md`, so the mission, principles, stack and invariants are not repeated here. How project knowledge is organized (layers, precedence, plan lifecycle): `project.md`, section “How project knowledge is organized”. `design.md` and `text.md` load on their own when the agent reads or edits the frontend and copy, and `process.md` when it reads or writes a plan.

## Where things are

- [`specs/docs/product-architecture.md`](../specs/docs/product-architecture.md) — **as-built architecture from the code**: runtime targets, models, services, sync flow. The layer closest to reality. Next to it: `specs/docs/product-ux.md` (behavior) and [`node-invariants.md`](../specs/docs/node-invariants.md).
- `specs/docs/dev-commands.md` — **the single source** of commands, environments, env variables and build traps.
- [`specs/work/`](../specs/work/) — what is in progress: `roadmap.md` (open phases), `observations.md` (debt registry), `plans/` (active plans), `phases/` (tails of phases 6–9).

## Operating rules

- **Branches = environments.** Work happens in `dev`; a push deploys staging `https://dev.overy.app` (+ demo `demo.dev.overy.app`). Prod `https://overy.app` is updated only through `composer run promote` (ff-merge `dev` → `main` + push). Locally: Herd `http://overy.test`.
- **Prod is Laravel Octane (FrankenPHP)**, a long-running worker. After a deploy or migrate: `php artisan octane:reload`.
- **Forge** is changed through **API v2** (`https://forge.laravel.com/api`, org-scoped; do not use `/api/v1`), except SSL certificates and changing a site’s repository: those are UI-only.
- **Desktop:** before `native:build`, stop the Vite dev server (otherwise `public/hot` ends up in the bundle: a white screen). Do not add patches in `vendor/`. Other traps: `dev-commands.md`, “Pipeline traps”.

## What exists

Thesis only. Open phases: [`specs/work/roadmap.md`](../specs/work/roadmap.md), structure: [`product-architecture.md`](../specs/docs/product-architecture.md), debt: [`observations.md`](../specs/work/observations.md).

- **Phases:** 1–3 + 3e (attachments) + 5 (production) done, 4 cancelled; 6 polish and 8 agent-loop partly done; 7 mobile and 9 E2E ahead.
- **Runtime:** prod `overy.app` (Postgres · Redis · Octane/FrankenPHP); desktop NativePHP (both flavors); one Laravel app. Local development and the default test run are Postgres too (`overy` / `overy_test`); the `sqlite` (desktop FTS5) and `budget` groups are kept out of the default.
- **Vault:** a graph of nodes and edges, a single label model (`labeled_by`), states ≡ groups + scope, sidebar tree, node list (sorting, multi-select and batch), Summary/Outline, graph view, DnD order, per-kind URLs.
- **Editor:** TipTap + markdown, mentions/tags, attachment-as-node, media and link OG.
- **Capture:** QuickCapture (web/desktop) and Telegram (text/voice/photo/PDF); uploads go through one seam, `AttachmentBlobStore` (compression before persist; the quota is the source of truth for size).
- **Agent:** MCP node control plane `/mcp/nodes`: 11 tools over the same actions as the web; every write takes a list of nodes; Sanctum ability `mcp`, tokens in Settings. There is deliberately no CLI.
- **Sync:** cloud sync and files-first (desktop chokidar); Crockford↔UUIDv7.
- **Cloud:** Filament admin and pricing plans; marketing (`/`, `/changelog`, `/legal`), public demo.
