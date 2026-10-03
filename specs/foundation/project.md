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
- **Design in code.** A new feature is built in the product right away: on its architecture, existing components and development data, without a separate mockup. The first version may be rough; after that it is refined, not rebuilt from scratch. A new UI primitive appears only when no ready-made one exists, and the reason is recorded in `specs/foundation/design.md`.

## How project knowledge is organized

| Where | What lives there | How it reaches the agent |
|---|---|---|
| `specs/foundation/` | Why the product exists and by which rules: `project.md`, `requirements.md`, `design.md` (design system), `text.md` (readers, tone, vocabulary), `process.md` (how work goes from an idea to the main branch) | `project.md` and `requirements.md`: in every session through Boost. `design.md` and `text.md` load when the agent reads or edits the frontend and copy, `process.md` when it reads or writes a plan (symlinks in `.claude/rules/`) |
| `specs/docs/` | As built: architecture, UX, node invariants, commands | By link |
| `specs/work/` | What is in progress: `roadmap.md`, the debt registry `observations.md`, active plans, open phases | By link |
| `specs/archive/` | Completed plans and closed phases | Not read unless asked |
| Overy vault | Ideas, tasks and questions for the owner | Through MCP |

**Precedence.** Foundation outranks the Boost guidelines in `CLAUDE.md`, skill rules, `.ai/rules` and plans: where they disagree, Foundation wins. If the project deliberately does not follow a Boost or skill rule, the reason is recorded in Foundation. If `docs/` disagrees with Foundation, or the code disagrees with `docs/`, that is drift, and it is recorded in `specs/work/observations.md`.

**Lifecycle.** A new plan is written in `specs/work/plans/`. When the plan is done, its outcome is added to `docs/` or `foundation/`, and the file moves to `specs/archive/`. Links inside the archive are not fixed. A finding that is not fixed right away is recorded in `observations.md` in the same pass. A closed entry is deleted from there.

**Editing the core.** After editing `project.md` or `requirements.md`, run `php artisan boost:update`, otherwise `CLAUDE.md` and `AGENTS.md` keep the old copy.
