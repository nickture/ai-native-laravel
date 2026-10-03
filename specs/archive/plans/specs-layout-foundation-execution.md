# specs: eternal, transient, archive + Foundation — plan

## Context

`specs/` is organized by GSD: PROJECT / REQUIREMENTS / ROADMAP / STATE, phases and plans. Work has long gone by need, not by phase number. So the eternal and the transient are mixed even inside one file. In `STATE.md`, 27 KB of key decisions sit next to a snapshot as of 24.08 and the history of sweeps. In `plans/`, about five of the 61 plans are live. The debt registry exists in two copies, a file and 88 nodes in the vault, and the two have already drifted apart.

The project has no Foundation, which both nickture skills refer to. There are no answers to the questions in `nickture-interface/rules/what-to-define.md`. Text decisions (tone, vocabulary, reader) live in the changelog header and in the agent’s memory.

The consultation on 02.10 (node `01m3yhfft1e3r9jxk0t3mfh2xz`) set out an approach, and this plan applies it to Overy. Foundation is kept apart from what Boost generates itself. Only a compressed core goes into every prompt, the rest loads by topic, and CLAUDE.md links to the other Foundation documents. Vendor components live separately, and everything of our own is built as wrappers. Tasks and ideas live in the tracker, and the agent triages them through MCP. Design is done in code.

Each session currently loads about 41 KB: `CLAUDE.md` (12.5 KB), `.claude/CLAUDE.md` (13 KB) and `MEMORY.md` (15 KB). The application code does not change in this pass.

## Decided

- **Eternal:** `specs/foundation/` and `specs/docs/` (an “as built” reference, read by link).
- **Transient:** `specs/work/`: active plans, the debt registry and open phases. Ideas, tasks and questions for the owner live in the Overy vault and are triaged through MCP. The 88 debt mirror nodes are archived; the file is the source of truth on debt.
- **Done:** `specs/archive/`, moved with `git mv`.
- **`STATE.md` is dissolved:** decisions go to their owners, the phase position goes to `work/roadmap.md`, the history of sweeps is deleted (it is in `git log`).
- **Foundation is a layer of four files:** `project.md`, `requirements.md`, `design.md`, `text.md`.
- **Loading: the core always, the rest by topic.**
  - The core is `project.md` and `requirements.md`. `.ai/guidelines/foundation.blade.php` pulls them in through `{!! file_get_contents(base_path('specs/foundation/…')) !!}`, and Boost inserts them into its block in `CLAUDE.md` and `AGENTS.md` on `boost:update`. `boost:update` itself runs on every `composer update`. Boost does not touch text outside its block (`GuidelineWriter.php:56`).
  - By topic: `design.md` and `text.md`. Each starts with a YAML field `paths:`, and `.claude/rules/` holds a symlink to the file. Claude Code loads such a rule when the agent reads or edits a matching file. Symlinks inside the project are supported.
  - `project.md` links to `design.md` and `text.md` so that Cursor and the nickture skills find them.
- **Code does not change.** Places where the code diverges from Foundation are recorded in `work/observations.md`.

## New layout

| Now | Where | Layer |
|---|---|---|
| `PROJECT.md` | `foundation/project.md` | core, always |
| `REQUIREMENTS.md` | `foundation/requirements.md` | core, always |
| — | `foundation/design.md` (front-end `paths:`: `resources/css/**`, `resources/js/**/*.svelte`, `resources/views/**`) | by topic |
| — | `foundation/text.md` (front-end `paths:`: `resources/js/**`, `resources/views/**`, `specs/**/*.md`) | by topic |
| `docs/*` | unchanged | reference |
| `STATE.md` | dissolve (M2) | — |
| `ROADMAP.md` | `work/roadmap.md`: open phases and their tails, the position from STATE | transient |
| `phases/06…09` | `work/phases/` | transient |
| `phases/01…05` | `archive/phases/` | archive |
| `observations.md` | `work/observations.md` | transient |
| live plans | `work/plans/` | transient |
| finished plans and `dogfood-legacy-import-execution/` | `archive/plans/` | archive |
| `plans/perf-probes/` | `scripts/perf-probes/` | tool |
| `~/.claude/plans/logical-juggling-sloth.md` | `work/plans/file-size-thinning-execution.md` (triage 4.1) | transient |

**A live plan** is a queue plan (`open-work-triage`, `next-quick-wins`, `product-forks-open-decisions`, `observations-fixes`, `ssot-audit`) or a plan with an open item that the triage or `observations.md` refers to. The rest go to the archive. Before the move, the list is shown as a table.

## Foundation: project.md

The vision and principles from `PROJECT.md` get two additions:
- **The “design in code” principle.** A new feature is built in the product right away, on its architecture, existing components and demo data, without a separate mockup. Then it is refined, not redone from scratch.
- **The “How project knowledge is organized” section.** Layers: `foundation/` is why and by which rules, `docs/` is as built, `work/` is what is in progress, `archive/` is what is done, the vault holds ideas, tasks and questions. Precedence: Foundation is stronger than skill rules, `.ai/rules` and plans, and a departure from a skill rule is recorded in Foundation with its reason. A mismatch between `docs/` and Foundation is drift, and it is recorded in `work/observations.md`. Lifecycle: a finished plan writes its outcome into `docs/` or `foundation/` and moves to `archive/`; links inside the archive are not fixed. The links to `design.md` and `text.md` live here too.

The GSD list in `.claude/CLAUDE.md` is replaced with a link to this section. The section is written so that someone can repeat the layout in another project from it: no secrets and no ties to the prod infrastructure.

## Foundation: design.md

| # | Item | Now | What is missing | Type |
|---|---|---|---|---|
| 1 | Brand and assets | The O_ mark with `--brand` (orange-500). Four logo variants, the “A.” favicon | UI principles, a source of truth for assets | record + ⚑ |
| 2 | Character on axes | Fragments (“like in Notion”, “everything is a type”, UX:213) | Six axes | ⚑ |
| 3 | Recognizability | Not recorded | A list of distinguishing features | ⚑ |
| 4 | Accessibility | No WCAG level | A minimum | ⚑ |
| 5 | Color | shadcn tokens and `--brand`. No success, warning, info, disabled, focus. Contrast not checked | Roles → tokens, the missing roles, a measurement of pairs | record + ⚑ + measurement |
| 6 | Layout | Tailwind defaults, breakpoints hard-coded in JS | Step, breakpoints, a rule for container queries | ⚑ |
| 7 | Typography | Overused Grotesk. Instrument Sans loads from the network and serves only as a fallback | Roles, scale, mono, loading | record + ⚑ |
| 8 | Elevation and layers | 14 different z-index values | A layer scale, a shadow rule | ⚑ |
| 9 | Radii | `--radius` and `sm/md/lg` | Role → radius | ⚑ |
| 10 | Motion | No tokens, 75–1000ms | Curves, durations, ceiling, reduced motion | ⚑ |
| 11 | Icons | Fluent via Iconify, the `fluentIcon.ts` / `NodeGlyph` registry | Default size, consolidation | record + ⚑ |
| 12 | Browsers | Not recorded anywhere. Electron 40, Vite `baseline-widely-available` | A matrix | ⚑ |
| 13 | Sandbox | The `/mocks/n` stub | Where it lives and what it shows | ⚑ |
| 14 | Tokens before components | Only in memory | The rule and its exceptions | record |
| 15 | Components | The shadcn files in `components/ui/` were edited in place (56 commits), and our own primitives (`Chip`) live there too | The rule “vendor code separate, ours as wrappers” | decided |

## Foundation: text.md

Items from `nickture-text-ru` (Foundation, glossary, `consistency.md:39`) and from what already exists in the header of `changelogReleases.ts:35-90` and in memory.

- **Readers and languages.** The interface and the changelog are for the user, in English. specs, plans and `CLAUDE.md` are for the owner and agents, in Russian. Commits are in English. There is no translation layer and none is planned. User content can be in any language, and the layout holds up with long Cyrillic titles.
- **Tone.** UI: the rules from the changelog header apply to the whole interface: present tense, “subject, verb, object”, no metaphors, name the result, not the mechanism, sentence case. specs: business style, current state only, no history and no rollbacks. Marketing: no kicker that repeats the heading.
- **Vocabulary.** node, note (only when not actionable), actionable (not task), label (not classify and not mention), type, state, Top (the root), Second Brain (capitalised in the interface, lowercase in running text), vault is an internal word only. “classifier” is not used in the interface; the contradiction in `product-ux.md:215,221` is checked against the code. A new word for the user appears only with the owner’s consent.
- **Formatting.** Headings in a node body are real `###`. The date and number format in the interface is taken from the code and recorded as is.
- **Changelog.** Its rules stay in the header of `changelogReleases.ts`, because whoever edits that file reads them. The word list there is replaced with a link to the vocabulary in `text.md`.

## Proposals for ⚑

Accepted together with the plan; edits go in the reply to the plan. The interview (step M3) asks only about the items that have no support in the code.

1. **UI principles:** progressive disclosure; one visual device means one thing (strikethrough means only “done”); a new action extends an existing surface instead of opening a new dialog; a picker’s action is the first item of its list; a frequent action has a key and a visible `Kbd` hint.
2. **Assets:** the O_ mark is a dark “o” and an underline in `--brand`. In the interface the source of the mark is `OveryMark.svelte`, files live in `public/icons/*`, sources in `resources/icons-source/`. The dev variant sits on a yellow tile. ⚑ Is the grey underline on the desktop tile intentional? Fonts and images are included through Vite from `resources/`.
3. **Character axes:** dense in navigation and lists, airy in the document body; types differ by glyph and label, not by layout; composability; conventionality (Notion, Obsidian, standard keys); directness (a metaphor only in the Top glyph); evolution.
4. **Recognizability:** the O_ mark and `--brand` as the only brand accent; Overused Grotesk; the NodeGlyph checkbox; chips with node titles; one icon system, Fluent; visible key hints.
5. **Accessibility:** WCAG 2.2 AA.
6. **Color:** a “role → token” table. Missing roles: success is green, warning is amber, info is blue (taken from the actual colors of `SyncIndicator`), disabled is `opacity-50`, focus is `--ring`. The user palette `colorCatalog.ts` and the priorities are a deliberate exception.
7. **Layout:** a 4px step, Tailwind breakpoints only for the page frame. A component inside a panel reacts to its own container through `@container`. The landing page uses `max-w-360`.
8. **Typography:** one typeface, Overused Grotesk; mono is the system stack. ⚑ Remove Instrument Sans? Body text is 16px, metadata is `text-sm`, `text-xs` is only for badges. Display tokens are only for marketing. In the editor the scale is in `em`.
9. **Layers:** base, raised, header, overlay, stacked dialog, toast, loader. A shadow appears only on floating surfaces, through `surface-panel`.
10. **Radii:** control and chip `md`, menu and popover `lg`, modal `2xl`, pill `full`; bare `rounded` is not used.
11. **Motion:** 150ms for press and tooltip, 200ms for menu and popover, 300ms for modal and sheet; 300ms is the response ceiling. Marketing is an exception. Curves: `cubic-bezier(0.23, 1, 0.32, 1)` for entering, `(0.77, 0, 0.175, 1)` for moving, `(0.32, 0.72, 0, 1)` for the sheet. Keyboard and frequent actions are not animated. Reduced motion by levels.
12. **Icons:** ⚑ default `size-5` with `-20-regular`, `size-4` only in compact controls. Names by meaning through `fluentIcon.ts`. Social network logos use `ph:*-logo`.
13. **Browsers:** ⚑ desktop is the built-in Electron; web is Baseline Widely Available plus the two latest major versions of Safari, iOS, Chrome, Edge and Firefox; anything beyond baseline is progressive enhancement only.
14. **Sandbox:** `/dev/states`, only in `local`/`testing`, `noindex`, replacing `/mocks/n`. Columns are states, rows are sizes. The page itself goes into `observations.md`.
15. **Components (decided):** shadcn-svelte in `components/ui/` is vendor code and is not edited in place. Our own styling and behavior are built as wrappers outside `ui/`. An existing component is used first. A new primitive appears only for a strong reason, and the reason is recorded in `design.md`.

## Design review (GRASP / SOLID / GoF / KISS / DRY / YAGNI)

No code is written; the lenses apply to how the documents split responsibility.

- **GRASP, Information Expert.** Each concept has one owner. Why and the rules: `foundation/`. As built: `docs/`. What the agent has in progress: `work/`. Ideas, tasks and questions: the vault. What was: `archive/` and `git log`. Token values: `app.css`. The block in `CLAUDE.md`: Boost. The decisions from `STATE.md` are spread across these owners instead of moving as one piece.
- **SRP.** Each Foundation file has one reason to change: vision, code requirements, design or text.
- **DRY / SSoT.** The copy of the core in `CLAUDE.md` and `AGENTS.md` is a build product; nobody edits it by hand. A symlink in `.claude/rules/` is not a copy. Duplicates of the vision and requirements in `.claude/CLAUDE.md` are deleted, otherwise the same text loads twice. After the move, the vocabulary in the changelog header, the memory files and the mirror nodes are deleted or archived. Links inside the archive are not fixed: the archive is not a source.
- **KISS.** One Blade shim assembles the core. Topic loading comes from the standard Claude Code mechanism: the `paths:` field and a symlink. Boost `@scoped`, with the files moved into `.ai/rules/boost/` and an index, was considered. Rejected: it needs a config flag, and the agent reads the index when asked, not on its own. A hook and a composer script that run `boost:update` on edit were considered. Rejected: `boost:update` already runs on `composer update`.
- **YAGNI.** This pass has no tokens, no component migration, no sandbox, no `.rgignore` for the archive and no screen audit against the checklist. All of this is recorded in `work/observations.md`.
- **OCP.** A new Foundation document is a new file in `foundation/` plus a line in the shim or a symlink. A new plan is a file in `work/plans/`. The structure stays the same.
- **GoF.** Composite was considered (the shim pulling sections in through `@include`). Rejected: Boost replaces `@include` with a placeholder (`RendersBladeGuidelines.php:33`), and `file_get_contents` over a list is simpler. The other patterns do not apply to documents.

## Steps

Before starting: reread `.ai/rules/index.md` and the `nickture-text-ru` skill (the documents are in Russian). Measure the size of what always loads: `wc -c CLAUDE.md .claude/CLAUDE.md MEMORY.md`. This plan is copied to `specs/work/plans/specs-layout-foundation-execution.md`.

**M1. Layout.**
1. Classify the 61 plans by the “live plan” rule and show the table.
2. `git mv` according to the “New layout” table. `docs/deep-dive/` (empty) and `.DS_Store` are deleted.
3. Link sweep across live files: `specs/foundation`, `specs/docs`, `specs/work`, `.ai/rules`, `.claude/CLAUDE.md`, three comments in tests (`ActionsAreFinalTest.php:24`, `CreateClassifierLeafTest.php:112`, `VaultNavBudgetTest.php:162`). Replace with `perl -pi` from an “old path → new path” table (BSD sed has no `\b`). The archive is not touched.
4. Commit `docs(specs): split eternal, work and archive`.

**M2. Dissolve `STATE.md`.**
1. Check each decision from “Key decisions (durable)” and from the sweeps against `foundation/requirements.md`, `docs/product-architecture.md` and `docs/product-ux.md`. If it is not there, add it to its owner: an invariant to requirements, structure to architecture, behavior to ux, words and design to text and design (in step M3). Descriptions of structure do not go into `requirements.md`, because it loads into every session. For the same reason, everything in it that describes structure rather than a rule moves to `docs/`.
2. “Phase position” → `work/roadmap.md`.
3. Delete `STATE.md`. Contradictions found during the check are fixed at their owner: Electron 38 versus 40, the node shape in the graph, “classifier” in UX, `apps/mobile` versus the one-app invariant.
4. Commit `docs(specs): dissolve STATE into its owners`.

**M3. Foundation: project, design, text.**
1. `project.md`: the “design in code” principle and the “How project knowledge is organized” section.
2. Interview: one or two batches of `AskUserQuestion` on the items with no support in the code: character axes, the desktop tile, Instrument Sans, icon size, browsers. The plan’s proposal comes as the first option.
3. Measurements. A script in the scratchpad converts the `app.css` tokens (hsl, oklch, `color-mix`) to sRGB and computes the WCAG contrast of key pairs in both themes. Separately, take a list of the `components/ui/` files that diverge from upstream shadcn-svelte (with the `diff` CLI if there is one, otherwise from the `git log` of each file after it was added), and a list of our own primitives inside `ui/`.
4. Write `design.md` (15 sections) and `text.md`, each with a `paths:` field at the start. Token values are not copied, only their names. The vision is given as a link to `project.md`. Each file stays under 200 lines. Run both through `nickture-text-ru`.
5. Replace the vocabulary in the header of `changelogReleases.ts:66-76` with a link to `text.md`.
6. Commit `docs(foundation): project principles, design and text`.

**M4. Loading, CLAUDE.md, registries, memory.**
1. Core: `.ai/guidelines/foundation.blade.php` → `php artisan boost:update`. By topic: symlinks `.claude/rules/design.md` and `.claude/rules/text.md` to the files in `specs/foundation/`. A line in `docs/dev-commands.md`: after editing `project.md` or `requirements.md`, run `boost:update`.
2. `.claude/CLAUDE.md`: replace the GSD list with a link to the “How project knowledge is organized” section. Delete what now comes from the core: stack, principles, invariants. Whatever is missing from `requirements.md` is moved there first. This file is edited last.
3. A new section in `work/observations.md`, “Code diverges from Foundation”: the “A.” favicon and the Filament brand, the dead `AppLogo.svelte`, different conditions for the dev icon, Iconify icons loaded from the network, Instrument Sans from Bunny, z-index, shadows that bypass `surface-panel`, 64 colors that bypass the tokens, no status and focus tokens, white on white in the dark `--sidebar-primary`, unused `--sidebar` and `--chart-*`, two default borders, graph colors in four places, breakpoints in JS, arbitrary font sizes in primitives, bare `rounded`, timings without reduced motion, no global `:focus-visible`, ESLint does not show Svelte a11y warnings, no sandbox, `components.json` points to a config that does not exist, contrast failures, edited vendor files and our own primitives in `ui/`. A separate entry: the screen audit against the `nickture-interface` checklist has not been done yet; it is the next pass after Foundation.
4. Vault: 88 mirror nodes under `01m0skbdm1e…`. Check each one against `work/observations.md`. If its content is in the file or the entry is closed, archive the node (`archive-nodes`). If the node holds something the file does not, move that into the file, leave the node alone and list it in the report. At the end, a report: archived, moved, skipped.
5. Memory: decisions that lived only in memory move to `design.md` and `text.md`, and the files are deleted together with their `MEMORY.md` lines: `feedback_semantic_color_tokens_only`, `feedback_text_sm_in_dialogs`, `project_chip_tab_geometry_size5`, `feedback_tailwind_canonical_utilities`, `feedback_actionable_vocab_word`, `feedback_mention_is_label_in_copy`, `project_root_label_top_marketing_second_brain`, `feedback_no_new_vocabulary_without_consent`, `feedback_no_overdesign_dialogs`, `feedback_picker_item_over_button`, `feedback_strike_means_done_only`, `feedback_vite_assets_over_public`, `feedback_no_kicker_echoing_heading`. In the other memory files the paths are updated (`specs/plans`, `STATE.md`, `REQUIREMENTS.md`, `observations.md`). `feedback_plans_location`, `feedback_claude_md_no_status_history` and `project_open_work_triage` (§4.9 resolved) are rewritten for the new layout.
6. Commit `docs(foundation): load the core through Boost, the rest by topic`.

Commits by path, no amend and no push.

## Verification

- No live file (`specs/foundation`, `specs/docs`, `specs/work`, `.ai`, `.claude`, `tests`, memory) links to the old paths: `grep -rn "specs/plans\|STATE.md\|ROADMAP.md\|specs/phases\|specs/observations.md\|PROJECT.md\|REQUIREMENTS.md"` is empty. Every relative link in live md files leads to an existing file (a script in the scratchpad).
- `git log --follow` on a moved file shows its history.
- After `boost:update`, `project.md` and `requirements.md` sit inside `<laravel-boost-guidelines>` in `CLAUDE.md` and `AGENTS.md`, `git diff` shows only the new section, and a second `boost:update` changes nothing.
- Topic loading works: in a new session `/memory` shows `design.md` after any `.svelte` file is read and does not show it before.
- The size of what always loads is measured before and after. It grew by no more than `project.md` and `requirements.md`, minus the duplicates removed from `.claude/CLAUDE.md` and the removed `MEMORY.md` lines.
- Every token name in `design.md` exists in `app.css`. Each of the 15 items has a section with an answer or with an explicit “not decided” and a reason.
- A search finds every decision from the deleted `STATE.md` at its owner.
- The vault report adds up: archived, moved and skipped nodes total 88.
