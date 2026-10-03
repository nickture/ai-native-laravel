# MCP control plane — bringing it up to the conventions

Goal: `/mcp/nodes` stops being a wrapper over the web views and becomes a surface designed for the agent. Input: what we learned walking 26 projects through the live server (what failed, what had to be guessed) + a review of the Notion / Linear / Obsidian MCP servers.

Reference point: `laravel/mcp` v0.9.3 ships Prompts, Resources, Completions, Pagination, structured content and annotations. Overy uses one primitive of the four: Tools.

## Start — where things live

M0–M4 and M7 are closed and verified on prod. Optional work remains: M5 (Resources / Prompts / MCP UI), M6 (deep traversal, orphans).

| What | Where |
| --- | --- |
| Tools (10) | `app/Mcp/Tools/Node/` |
| Payload projectors | `app/Mcp/Support/` — `NodeSummary` (echo) · `NodeDetail` (get-node) · `NodeListing` · `NodeSearchResults` · `BatchNodes` (write envelope) · `RowKey` (row id). Each one also declares its own `::schema()` |
| Server instructions + tool registry | `app/Mcp/Servers/NodeServer.php` |
| View vocabulary (SoT of the `view` description) | `app/Enums/VirtualView.php` — 12 values |
| View → internalView resolution | `app/Services/Vault/Routing/VaultKeyResolver.php` |
| Container × dimension intersection | `app/Actions/Vault/LoadVaultPage.php` — the `$scope` parameter; `RenderScopedList` only maps the web dimension to a view key |
| Edges in both directions | `app/Services/Graph/Backlinks.php` — `for()` / `outgoing()` |
| Search filters | `app/Services/Search/FilterParser.php` — `parse()` + `rejectedFilters()` |
| Tests | `tests/Feature/Mcp/` |
| Package primitives | `vendor/laravel/mcp/src/Server/` — `Completions/EnumCompletionResponse`, `Pagination/CursorPaginator`, `Prompts/`, `Resource.php`, `Concerns/HasStructuredContent` |
| User-facing description of the surface | `resources/docs/mcp.md` (SoT) · changelog `resources/js/lib/changelogReleases.ts` |
| Rules | `.ai/rules/mcp.md` |

Branch `dev`. Gates: `vendor/bin/pint --dirty --format agent`, `vendor/bin/phpstan analyse --memory-limit=2G --no-progress`, `php artisan test --compact --parallel` + a serial run of the changed scope.

## Prod verification status — all green

Prod (`overy.app`, commit `6772253e`) was verified with live calls through curl. The MCP client caches the tool list from the moment it connects, so it cannot be used to check itself after a deploy.

1. **`RowKey`.** `search-nodes trait:classifier` returns bare 26-character keys; the classifier key `01kzxaemkgefsst97men5zqjz7` (“lum”) opened in `get-node`. Before, it came as `classifier:{grouper}/{slug}` and the tool did not accept it. A view hit comes as `graph`, not `view:graph`.
2. **Surface.** Exactly 10 tools, on one page (the 19 used to overflow and get cut by the cursor). `readOnlyHint: true` on the three reads, `openWorldHint: false` on all, `destructive`/`idempotent` on every write, `outputSchema` on all ten.
3. **Slice and pages.** Project X (14 children): `inbox ∩ X` = 10, `done ∩ X` = 4, exactly the census of states inside the container; `next ∩ X` = 0 against 43 globally, an honest zero. The cursor `eyJvZmZzZXQiOjN9` led to the second page, and `total` holds at 1954. A row carries `traits` and `updated_at`: in the `attachments` view the very first row is titled “A new guide in our series…” and is recognizable as an attachment ONLY by its trait. That is exactly the blindness the field was added to fix.
4. **Promotion.** A note without a trait got `traits: ['actionable']` after `set-state next` and appeared in the `next` queue. The answer came in the `{nodes: [...], skipped: []}` envelope with the full node.
5. **Texts.** `/docs/mcp` returns 200 with the new tables (`move-nodes`, `label-nodes`, `archive-nodes`, `update-nodes`), the old `/docs/mcp-cli` returns 404. The landing page has no CLI card. The built changelog chunk has all seven of today’s entries and the 43 badge.

The check node `01m07mr6cxe6c8rzj8p8dn1w8e` was archived right after check 4.

## What we take and what we don’t

Applicable, we take:

- **Linear — flat filter parameters** (`assigneeId`, `stateId`, not a nested GraphQL object). Maps directly onto our main gap.
- **Linear — parameter values in the tool description.** The agent should not have to guess the vocabulary; we have 12 views, and the description names six of them.
- **Linear — curation by job, not by endpoint.** Their 23 tools are NOT “few”; they are a selection.
- **Notion — the search + fetch pair as two read primitives.** We already have this (`search-nodes` + `get-node`), which confirms the layout is right.
- **Obsidian — graph traversal with direction and depth, orphan detection.** Native to a second brain; our depth is exactly 1 and orphans are invisible.
- **The common critique of Linear — stringified JSON in `text`.** We do exactly this, and with `JSON_PRETTY_PRINT` on top.

Not applicable, we don’t take:

- **Notion semantic search** — tied to their own AI layer. We have FTS + fuzzy; that is a separate product conversation, not an MCP job.
- **Notion page-level fetch/replace of the whole page** — our granularity is finer (node, edges, meta); falling back to “the whole page” would be a regression.
- **CSV/TSV instead of JSON** — proposed for wide tables; our rows are narrow (7 fields), and the loss of nesting eats the gain. Structured content gives the same effect.
- **Tool count as a goal.** Linear has more tools than we do. We collapse tools not for the count, but because ours are mechanical twins (a single and a batch version of the same action).

---

## M0 — discoverability — done

1. **The `list-nodes` description is built from `VirtualView` + `StateSlug`** in `description()` instead of sitting as a literal in `#[Description]`: the vocabulary belongs to the enums, and the copy had already gone stale (6 views of 12). Both open forms are named: a classifier slug and the key of a node itself (its children). The second one used to be found only by trial and error.
2. **Annotations on all 19 tools.** `readOnlyHint` on the three read tools, `openWorldHint: false` everywhere (no calls to the outside), `destructiveHint` / `idempotentHint` on every write. The client can now auto-approve reads and knows whether a retry is safe.
3. Tests: `tests/Feature/Mcp/ToolDiscoverabilityTest.php` (vocabulary + annotation matrix) and a lock on view-by-node-key in `ListNodesToolTest`.

**Completions for a tool are impossible. This is not our gap but a protocol boundary.** `completion/complete` accepts only `ref/prompt` and `ref/resource` (`Server/Methods/CompletionComplete.php::resolvePrimitive`); tools have no autocompletion in either the spec or the package. The equivalent for a tool parameter is a vocabulary in the schema. `view` is open (the user’s own states, classifiers, node keys), so an `enum` would lie, and the listing in the description stays the truth. Real completions become available in M5 together with Resources/Prompts: there `view` in a URI template gets them.

## M1 — the exact question — done

`list-nodes` got `container`, a node key that pins the view to that node’s contents. `view` picks the dimension, `container` picks the container. “What is in progress inside project X” became one call instead of 26 lists that lose the binding.

**There was nowhere to extract the intersection to: it was already shared.** `LoadVaultPage::handle()` has accepted `?Node $scope` from the start, and `RenderScopedList` does not own the logic; it only maps `(dimKind, dimKey)` to a view key. `NodeListing` now passes the scope into the same call. Parity with the web is locked by a test: rows from the tool and rows from `vault.scoped` are compared by id and order.

The other flat filters from the original list (`state`, `priority`, `label`, `due_before`, `updated_after`) did not need to be added. They are the `view` dimensions: state, `prioritized`, a classifier slug, `overdue` / `today` / `this-week`, `aged` / `recent`. A filter language of our own on top of them would be a second answer to “what belongs to the slice”.

**Not taken — backlinks as a filterable list.** They turned out not to be free: the web pins them through `scopeIds` + its own view key (`labeledby:{id}`), which means two more branches. And `get-node` already returns both sides of the graph in full.

**Fork closed: `set-state` promotes to actionable.** Both call sites (`set-state`, `batch-set-state`) moved to `PlaceNode`, the same Command that sits behind the web drag-into-queue. The web keeps two gestures: join a state as a member (state-PATCH) and drop a node into a queue (`/place`, promotes). The control plane has one state tool, and “set-state: next” from an agent means “make this a next action”. Before, it called the non-promoting path, and the node settled in a state that its own queue does not show (queues list only actionables). Promotion was not rewritten; it was reused.

## M2 — an honest row — done

1. **`traits` in the row, for both listing and search.** Not `kind`: `VaultKind` tells a classifier from a node, but not a screenshot from a note, and `kind` in a search row already has another meaning (the type of the ROW: `node` is a record for `get-node`, `view` is a view for `list-nodes`). Traits are the SoT itself; both row builders already collected them and dropped them on output.
2. **`updated_at` in the row**, for both projectors.
3. **A cursor instead of `truncated: true`.** `Pagination/CursorPaginator` over the already materialised set; rows are projected AFTER the slice, so only the window pays. The answer carries `total` (the whole set) and `nextCursor` while rows remain; an unreadable cursor means the start, not an error.

## M3 — the wire — done

`Response::structured()` does both at once: it puts `structuredContent` next to the text and encodes compactly. So a separate “drop pretty-print” step was not needed. Pretty-print went away together with the manual `json_encode`, and `NodeSummary::JSON_FLAGS` went with it.

`outputSchema` on all ten. The schema is declared by whoever builds the payload: `NodeSummary::schema` / `NodeDetail::schema` (extends the summary exactly the way its payload extends the summary payload) / `NodeListing::schema` / `NodeSearchResults::schema` / `BatchNodes::schema`. The declared and the returned have no place to drift apart.

`handle()` now returns `Response|ResponseFactory`: `structured()` returns a factory, `error()` stays a Response.

Side effect in tests: eight assertions matched the string `"key": value` from the pretty format. They were rewritten to parse the payload. They should have looked at the data, not at the indentation, from the start.

## M4 — the surface — done

19 → 10. Every write tool takes `nodes`, whether one node or fifty; a separate “batch” no longer exists as a concept.

| now | before |
| --- | --- |
| `set-state` | set-state + batch-set-state |
| `move-nodes` | move-node + batch-move |
| `label-nodes` | add-label + remove-label + add-type + remove-type + batch-label |
| `archive-nodes` | archive-node + unarchive-node + batch-archive |
| `update-nodes` | update-node + batch-meta |

The read trio and `create-node` / `split-node` are untouched.

One answer envelope for all writes (`BatchNodes`): `nodes` holds the changed nodes IN FULL, `skipped` says what was skipped and why. Not bare ids: a write that names only ids forces a second read call to see WHAT it did, and that is why the single tools lived next to the batch ones. A node the operation did not change appears in neither list.

Arguments that name the value of ONE node (`title`, `content`, `before`, `after`) are refused when given a selection; otherwise one title would overwrite every row. One refusal serves them all: `BatchNodes::singleNodeRefusal`.

Key extraction and the no-op guard also moved into one place: `BatchNodes::keysFrom` instead of five copies of the same `array_filter`, and `NodeBatchOps::fields`, an extension of `meta()` to the node’s columns. So title/content inherit the same “did it change” check as the meta scalars, and the web batch keeps calling `meta()` without changes.

## M5 — primitives

1. **Resources**: `overy://node/{id}`, `overy://view/{slug}`. The user pins a node into context themselves, without the model calling a tool. Subscriptions: the client learns about a change instead of asking again. **Completions** appear here too (`EnumCompletionResponse` over `VirtualView`): they work on a URI template, and on a tool parameter they do not exist.
2. **Prompts**: `weekly-review`, `process-inbox`, `triage-overdue`. Today every agent invents the procedure anew; with 50+ overdue nodes this is not theory.
3. **MCP UI (`AppResource`)** for the graph and Summary: return an interactive page instead of a wall of text. It goes last: the largest volume, the lowest urgency.

## M6 — native to a second brain

1. **Traversal deeper than one step**: direction (outgoing / incoming / both) + depth, BFS. Today `get-node` gives exactly one step.
2. **Orphans**: nodes with zero incoming links. Cheap on top of the same reverse query, and it is exactly the question a second brain should be able to ask itself.

## M7 — what a human reads — done

- **`resources/docs/mcp.md`** (slug `/docs/mcp`, formerly `/docs/mcp-cli`, renamed together with dropping the CLI promise) is also the SoT of the public surface. The tool tables are rewritten for the ten, and the page now covers `container`, the cursor, traits in the row, both sides of the graph, the answer envelope and the refusal of single-node arguments on a selection. The claim “putting a node in a state never makes it actionable” was plainly false and is replaced.
- **Changelog** (`resources/js/lib/changelogReleases.ts`): four entries in features and three in fixes for 13 August; the day’s commit counter raised from 21 to 38.
- **Landing page**: nothing to touch. MCP is described there at the level of “Overy speaks MCP, point your agent at it”, without tool names or counts.
- **Settings → MCP & integrations**: the same. It lists verbs (“search, create, edit, move, set state…”), not tool names. Correct as it was.
- **`specs/work/observations.md` W0.5**: the reversal of the decision, with its reason, is appended to the closed entry; the module row in the table is fixed.
- **`specs/docs/product-architecture.md` §11b, `.claude/CLAUDE.md`**: the counter goes 19 → 10, “batch” as a concept is removed, state-as-folder is replaced with promotion.

**Found along the way, not touched:** the landing page and docs promise a CLI (“on the way”, a card in upcoming), while `.claude/CLAUDE.md` states “no CLI, on purpose”: an Artisan wrapper was written and then removed so as not to keep a second surface. One of the two is untrue for the reader. This is a product decision, not mine.

---

## Design review

- **Information Expert.** The view vocabulary belongs to `VirtualView`, the intersection to the scoped-list service, the row key to `RowKey`. None of them is duplicated in the MCP layer; MCP only projects.
- **SSoT.** M1 must extract the intersection into a shared service, not reinvent the filter in `NodeListing`. Otherwise a second answer to “what belongs to project X” appears, and it drifts from the web exactly the way the `canonicalKey` spellings drifted.
- **Open/Closed.** Flat filters extend by adding a field; named views stay shortcuts on top, not the only entry point.
- **Breaking changes** are concentrated in M4 and ship in one wave, not spread across M0–M3.
- **Each wave is its own commit with tests.** Gates as usual: Pint, PHPStan L10, a full run + a serial check of the changed scope.

## Already done (outside the waves)

Edges in both directions on `get-node` (`references` / `backlinks`), one Crockford format on the wire (`RowKey`), a loud refusal on a filter that cannot run. Rules: [`../../../.ai/rules/mcp.md`](../../../.ai/rules/mcp.md).
