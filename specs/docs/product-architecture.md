# Overy — product architecture (reverse-engineered from code)

> This document was reconstructed **from code** (`app/`, `routes/`, `database/migrations/`), without cross-checking against the specs. The goal is a mid-detail “map of the territory”: which layers exist, who is responsible for what, how the lifecycle of key objects and requests runs. If code and spec diverge, this records the code as of 2026-07-21.

> **Excerpt.** This sample keeps sections 1–3 and 11b. The full document is about 236 KB.

---

## 1. What it is, and three runtimes

Overy is a local-first note store: content lives as a graph of **nodes** (`nodes`, markdown documents) and **edges** (`edges`, directed links). The same Laravel app runs in three modes, and almost every fork comes down to two questions: **who owns the data** and **is there a disk/network**.

| Mode | What it is | Auth | `NodeMirror` | `VaultEventSource` | Owner (`user_id`) |
|---|---|---|---|---|---|
| **web** | browser on the cloud server | Fortify (session) / Socialite | `NullNodeMirror` (no disk) | `NullVaultEventSource` | UUID of a real `User` |
| **desktop-anonymous** | NativePHP app without a link | guard `overy-local` → `LocalOwner` | `DesktopNodeMirror` (writes `.md`) | `ChildProcessVaultEventSource` (chokidar) | `LocalOwner::ID` (fixed UUID) |
| **desktop-linked** | NativePHP after linking to the cloud | `overy-local` + Sanctum token for sync | `DesktopNodeMirror` | `ChildProcessVaultEventSource` | `DeviceCredential::current()->user_id` |

One class knows the runtime fork — **`Support\Environment`** (`isDesktop()` / `isWeb()` from env/NativePHP). The owner fork is known by **`Support\OwnerResolver::resolve($request)`**, in this order:

1. the authenticated `User` (web), if its id ≠ `LocalOwner::ID`;
2. on desktop — `DeviceCredential::current()->user_id` (the linked cloud);
3. otherwise — `LocalOwner::ID`.

Controllers call `OwnerResolver` through the base `Controller::ownerId()` helper. **Important nuance:** sanctum routes (`/api/sync/*`, `/api/auth/me`) deliberately use `authUserId()`, not `ownerId()` — otherwise the fallback to `LocalOwner` would silently swap the owner on an auth failure.

**The `users` table is created in every runtime** (on anonymous desktop it is empty, at no cost): after a cloud link, desktop sync looks up the cloud user by id, and the former desktop gate on the table broke every post-link `sync:tick` with `no such table: users` → the pull of web-origin blobs never ran.

**There are two cloud environments, both on one server:** prod `overy.app` (+ demo origin `demo.overy.app`, Octane `:8001`) and staging `dev.overy.app` (+ `demo.dev.overy.app`, Octane `:8002`). The same app, with different branches (`main` / `dev`), databases, Redis prefixes with a separate ACL user, and S3 buckets; isolation is by namespace, not by instance. What exactly an environment needs besides the site — `specs/docs/dev-commands.md`.

**Prod web — Laravel Octane (FrankenPHP).** The cloud (`overy.app`) is served by a long-running Octane worker (`laravel/octane`, the driver is chosen via `OCTANE_SERVER=frankenphp`; the stock config default `roadrunner` is left alone, the binary/worker is gitignored — Forge installs it, nginx proxy + supervisor). Dev/desktop — plain php-fpm/`serve` (the container boots per request). The key consequence: Octane boots the container **once** and reuses it across requests, so all per-request/per-owner memoizers that under php-fpm were in effect per-request `singleton()` are rebound as **`scoped()`** (`Container::forgetScopedInstances()` clears them before every request but shares them within one — keeping one bulk load per visit) in three providers (§2): `GraphServiceProvider` (`EdgeRepository`/`EdgeAttachmentService`), `VaultServiceProvider` (read path: `VaultKeyResolver`/`NodeFinder`/`CategoryTreeAssembler`/`TypeHierarchy`/`StateSlugResolver`/`StateMachine`/`RecentNodesResolver`/`RecentVisitStore`/`VaultListQuery`/`ClassifierIndexPresenter`) and `AppServiceProvider` (`RecentMirrorWrites`/`UserUsageStats`/`DefaultTemplateApplier`/`LabelKindResolver`). Without `scoped()`, request 1 would hand its graph/counter cache to every later request of the worker (stale Summary/sidebar + cross-owner leak). Guard — `OctaneScopedBindingsTest`.

---

## 2. Layers and control flow

A classic Laravel layering with an explicit actions/services split:

```
HTTP (routes/*.php)
   → Controller            thin: resolve owner, validation, mapping to DTO, presentation
      → Action (app/Actions)   write scenario (CreateNode/UpdateNode/MoveNode, Telegram, Uploads)
         → Data (app/Data)     immutable DTO, framework-agnostic input for the action
         → Service (app/Services)  domain logic (Graph, Classifier, Outliner, Sync, Vault…)
            → Model (app/Models)   Eloquent: Node, Edge, Sync*, Device*, User
```

- **Controllers** hold no domain logic. `VaultController` is thin — it delegates to collaborators (`Actions\Vault\RenderScopedList`, `NavShellPresenter`, `Services\Vault\Routing\VaultRouteRedirector`, `Support\Vault\VaultListWindow`); `NodeController` validates and delegates to actions.
- **Actions** (`final readonly`) are the unit of writing. One public `handle()`, everything through constructor DI.
- **Data/DTO** (`CreateNodeData`, `UpdateNodeData`, `TranscriptionResult`, `VaultRoute`, `ViewContext`) are immutable, with `fromArray()` factories; they decouple the action from the shape of the `Request`. `UpdateNodeData` carries a `present[]` map (it tells “field not sent” from “sent null = clear”).
- **Contracts** (`app/Contracts`) — interfaces for substitution points: `EdgeRepository`, `EdgeInvariant`, `ViewFilter`, `TemplatePropagator`, `VaultEventSource`.
- **Composition root** — `Providers\AppServiceProvider::register()` (delegates the read-path scoped bindings to `GraphServiceProvider` + `VaultServiceProvider` and registers them from here — §1). Interfaces and implementations are bound here; this is where `Environment` is asked, so that domain code stays runtime-agnostic. **Tagged bindings** are used for three strategy chains (registration order = execution order):
  - `graph.invariants`: `SelfEdge → TargetTrait → RequiredSourceTrait → SourceTrait → Cycle → LabeledByCycle`;
  - `classifier.propagators`: `Fields → Traits → State`;
  - `vault.view-filters`: `Classifier → Type → Node → Special → Group`.

---

## 3. Domain model: nodes, edges, traits, predicates

### Node (`app/Models/Node.php`)
- `HasUuids` + `SoftDeletes`; mass assignment through the `#[Fillable([...])]` attribute.
- `parsed_meta` (JSON) is cast through `ParsedMetaCast` — it is the “pocket” for everything not moved out into columns: `traits[]`, `slug`, `plural`, `fields{}`, duplicate edge scalars (`parent_id`, `now_in`, `extends`, `classified_by[]`, `scoped_to[]` — all written by ONE seam, see §5), order mirrors of manual order (`parent_position`, `sidebar_order`, `state_position`, `classifier_positions{slug: int}` — LWW scalars that travel through sync/frontmatter as ordinary meta fields, see §5a) and `perFieldUpdatedAt{}` (sync stamps). Eloquent re-resolves an array cast on every property read, so `Node::getAttributeValue()` memoizes the `parsed_meta` decode by the raw string — JSON is decoded once per value.
- `born_in` → enum `BornIn` (int-backed, provenance — where the node was born, **immutable**, stamped only on INSERT): `Undefined=0`, `Legacy=1`, `Filesystem=2` (manual create / copy / external import, set by `FileImporter` on INSERT), `Seed=3`, `Desktop=4`, `Web=5`, `Mobile=6`, `Telegram=7`, `Mcp=8`. A write through MCP is born with `born_in=Mcp` on every path: nodes from `create-node`/`split-node` and a classifier created by `label-nodes` with `create: true` (`ClassifierByName::classifierFor(..., bornIn: BornIn::Mcp)`).
- `touched_in` → the same `BornIn` enum, but the **mutable twin** of `born_in`: the channel of the LAST edit, and its contract is agreement with `updated_at` (once the row says it changed, it also says where from). The stamping hangs on `Node::updateTimestamps()`, so every save gets it, including `saveQuietly()`; a caller that took the row’s clock for itself (`updated_at` by hand) or named the channel itself is not overridden. On INSERT it mirrors `born_in`, so a seeded row does not start life claiming that someone touched it. An edit that lived only in the graph (placing into a state, a label — edges and their `parsed_meta` mirrors are written quietly, by design) is stamped by `Services\Node\RowChangeStamp`: a snapshot of meta before the mutation, a stamp after, and only if meta actually changed and nobody moved the clock. Without it, an agent’s day of filing moved neither `recent` nor the pull cursor `(updated_at, id)`.
- `born_by` / `touched_by` (varchar 120, nullable) — the **actor** next to the channel: who stood between the owner and the write. For MCP it is the name of the token the client authenticated with — the only identity on the request (the handshake `clientInfo` does not work: laravel/mcp keeps no session store, and its session id resolves to `''` for a client without the header — one key for all users at once). Channel and actor are set by ONE setter, `WriteChannel::declare(channel, actor = null)`, so the channel cannot update with a stale actor next to it; on INSERT `born_by` is copied into `touched_by` exactly as `born_in` into `touched_in`. Hand setters that took the row’s clock must name both (`RowChangeStamp` — the path of any MCP state/label write; `FileImporter` — deliberately nobody). The stamp guard asks “did the caller take the clock”, NOT “did `updated_at` move”: the column stores seconds, and a second write within the same second looks clean to Eloquent (create + label as two MCP calls in a row kept the first channel). The model’s ceiling: one token pointed at two agents reads as one agent; the fix is naming the token. MetaPanel draws “15m ago · MCP · Claude Code”. The multi-user form (`*_by_user_id` for “mine vs someone else’s”) waits for sharing — observations.
- **Pristine-seed detection:** `PRISTINE_HASH` (all-zeros sentinel) + `isPristineSeed()` (`born_in=Seed && hash=zeros && live`) + scope `excludingPristineSeeds()`. The untouched built-in vocabulary is identical on every device by slug → it does not need syncing (see §10). The first real edit recomputes a genuine hash → the customized vocabulary syncs again.
- **`booted()` hooks:**
  - `forceDeleted` → `Cascade::onNodeForceDelete` (scatter edges only on hard delete; soft delete leaves the node in the graph) — for `attachment`-trait nodes, **before** the edge cascade it runs `AttachmentEmbedStripper::strip` over all hosts, so that `![](id)` / `<img src="id">` in other nodes’ bodies do not remain a broken image;
  - `deleted` → `AttachmentBlobReaper::reap` (remove the blob/sidecar), then `VaultLayoutSynchronizer::afterChildDeleted` (demotes a container parent that became childless, on **any** delete path — controller / sync pull / cascade);
  - `restored` → `VaultLayoutSynchronizer::afterChildRestored` (the reverse operation: undo / sync round-trip re-promotes the host back into a folder);
  - `saving` → `ensureClassifierSlug` (auto slug from title for classifier nodes) + `syncSearchText` (maintains the device-local column `nodes.search_text` = `PlainText::of(content)` — text without HTML tags and image/link URLs; the FTS5 triggers and PG ILIKE target it, so an image `src` does not match in body search; the column is derived — not in the `NodePusher` allowlist and not in `NodeHasher`, it does not travel over the wire) + `syncTitleNormalized` (a second derived column of the same class: `nodes.title_normalized` = `MentionLabel::normalize(title)`, index `(user_id, title_normalized)`; through it `MentionResolver` matches a bare `[[Title]]` with `whereIn` instead of reading the WHOLE vault on every mention-bearing write — 132.8 ms → 6.6 ms on 4309 nodes, and flat as the vault grows. `normalize` has no portable SQL form (it collapses wikilink syntax: `Backup Brain [[Thiago Forte|id]]` → `backup brain thiago forte`), so a `LOWER(title) LIKE '%needle%'` prefilter is unsafe — the needle is not contiguous. Writers: Eloquent paths through the hook, `DemoVaultRestorer` (bulk insert bypassing the model) derives it itself, the migration backfills including trashed rows — the ghost contract resolves links to soft-deleted nodes).
- The `content` attribute canonicalizes `'' → null` on write (a single source of truth for nullability).
- `resolveRouteBinding` decodes a 26-character Crockford-Base32 key back into a UUID; anything not in canonical form → 404.
- `kind()` is derived (Type/Instance/Note), computed on the fly through `NodeKindResolver` (not stored).
- **`Support\WriteChannel` (scoped) is the only owner of the write channel.** `current()` = the declared channel, otherwise `BornIn::runtime()`; `actor()` has no fallback (null = the owner themself). An entry point that is itself a channel declares it for the request with `declare()` (`NodeServer` in its constructor — `BornIn::Mcp` + the token name); a unit of work uses `during()` (`FileSyncService` — `Filesystem` per watcher event, `PullApplier`/`NodePusher` — the peer’s channel per sync row). It is read by `Node::updateTimestamps()` and `RowChangeStamp`. Telegram does not declare a channel: `TelegramInboxCapture` sets `BornIn::Telegram` directly in `CreateNodeData`.

### Edge (`app/Models/Edge.php`)
- `HasUuids`; `#[Fillable(from_id, predicate, to_id, position)]`; the `predicate` column (SMALLINT) is cast to `EdgePredicate`. (The column used to be called `type` — renamed because `type` collided with the node trait `type` and with the field types `link`/`union`; semantically it is a predicate.)
- **No FK** on `from_id`/`to_id` (as on `nodes.user_id`) — integrity is held by the application-level `Services\Graph\Cascade`. This is a deliberate project decision.
Postgres-only DDL: CHECK `edges_predicate_check` (whitelist of live `EdgePredicate` numbers), GIN index `nodes_parsed_meta_fields_gin` on `parsed_meta.fields`, the `parsed_meta` column is JSONB. SQLite has none of this; covered by `tests/Feature/Schema/EdgesPostgresSpecificTest.php` in the default (Postgres) run.

### EdgePredicate (`app/Enums/EdgePredicate.php`) — 5 live predicates, int-backed (numbers frozen forever; 3/5 retired)

| Predicate | int | Semantics | Cardinality | Required target trait | Cycle protection |
|---|---|---|---|---|---|
| `ChildOf` | 1 | outline tree | **Single** | — | yes |
| `Extends` | 2 | type inheritance | Single | `type` | yes |
| `NowIn` | 4 | pointer to the current state/group | Single | `state` | no |
| `ScopedTo` | 6 | classifier scope (a group/tag is available only in the target + its descendants) | Multi | — (any target) | yes (`CycleInvariant`) |
| `LabeledBy` | 7 | **single labeling axis** (tag/type membership **∪** `[[wiki]]` mention) | Multi | — (the target decides the nature) | per-target (`LabeledByCycleInvariant`) |

**`LabeledBy` = merger of the former `ClassifiedBy`(3) + `Mentions`(5)** (M3 sweep 2026-07-03). One predicate carries EVERY labeling edge; **the semantics are decided by the nature of the TARGET** through the single seam `Services\Graph\LabelKindResolver::of` → `Enums\LabelKind`:
- the target carries `[classifier]`/`[type]` → **`ClassifierMembership`** (classification/membership);
- a `[state]` target → **`PlainReference`** (“state = mention only”, §7a — a state never classifies, it is addressed by `now_in`; checked before the classifier test);
- otherwise → **`PlainReference`** (a plain mention).

The retired predicate strings `'classified_by'`/`'mentions'` **survive as an FE presentation vocabulary** (graph edge styles, mention chips), but their SoT is `LabelKind::wireSlug()` (`ClassifierMembership`→`classified_by`, `PlainReference`→`mentions`), not `EdgePredicate::slug()`. `fromSlug('classified_by'|'mentions')` throws, `tryFrom(3|5)` = null; the pgsql CHECK is narrowed to `IN(live)`, so 3/5 will not come back. `LabeledByCycleInvariant` is a cycle guard **only for the membership subset** (a mention cannot fabricate a membership cycle; batched node loads, no N+1); the generic `CycleInvariant` holds `child_of`/`extends`/`scoped_to`.

`ScopedTo` (§7a) is the only predicate with a requirement on the **source trait** rather than the target: the source must carry `[classifier]` (`requiredSourceTrait`, checked by `RequiredSourceTraitInvariant` in §6), the target is any node. **`ChildOf` is now Single** (M2 sweep): a node has ≤1 outline parent (attachments included; the multi-host carve-out was aspirational and no live path created it; a forward migration collapsed legacy multi-parent into the min-id parent, on PG via `MIN(CAST(id AS TEXT))`, since there is no `min(uuid)`). Slug strings (`child_of`…`labeled_by`) are wire format only (frontmatter + Inertia); round-trip through `slug()`/`fromSlug()`.

### NodeTrait (`app/Enums/NodeTrait.php`) — 5 traits, two categories (`TraitCategory`)
- **classifier-side** (definitions): `classifier`, `state`, `type`;
- **instance-role** (content): `actionable`, `attachment` (`contact`/`agent` retired — MVP scope; `attachment` is auto-assigned, not in the picker).
- `state`/`type` implicitly imply (`implies()`) `classifier`. The categories are mutually exclusive (`conflicts()`), validated by `NodeInvariantValidator`. **The nature of a node is losslessly reversible** along the ladder note ⇄ `[classifier]` tag ⇄ `[classifier,type]` type: toggling traits does NOT touch `labeled_by` edges (membership vs reference is a read-derived `LabelKind` from the target’s trait), so “Make this a / not a type/classifier” round-trips without edge churn (§7).

### Dual representation (an important cross-cutting pattern)
Structural links live at once as **edges** (UUID-stable, the source of truth for the graph) and as **mirrors in `parsed_meta`** — scalars (`parent_id`, `now_in`, `extends`) and lists (`classified_by[]`, `scoped_to[]`). All of them are **UUIDs** (stable identity): `classified_by` is NO LONGER a slug list — the slug collided between nodes with the same name (context “Home” and project “Home” = `home`), and membership resolved/detached to the wrong node; `extends` moved to uuid next.

The owner of the mirrors is **the edge write seam `Services\Graph\EdgeAttachmentService`**, the only point that knows an edge changed. It already keeps the post-write state coherent (the layout synchronizer), so the mirror is the same responsibility: on every `attach`/`detach`/`detachAllOutgoing` it writes the predicate’s scalar mirror (`EdgePredicate::scalarMirrorKey()`) and rebuilds the list mirror (`listMirrorKey()` → `ListMirrorWriter`) from the edges it wrote itself. The write mechanics are `EdgeMirror` (`writeId`/`writeIds`/`rebuild`, all through `StampedMetaWriter`: quiet save + per-field LWW stamp); the nature filter for `classified_by` (the `LabelKind::ClassifierMembership` subset of `labeled_by`) and the union of manual plain labels are `ClassifiedByCacheWriter`. No creation path — web, desktop, sync receive, seeder, import — can produce an edge without a mirror; the rebuild from edges also self-heals a row whose cache diverged earlier. **A manual plain label** is the only part of `classified_by` not derived from edges (a plain edge does not describe itself as “attached by hand, not derived from the body”), so its subset is written by `NodeEdgeApplicator`, and the rebuild merges it in.

**Naming at the persistence boundary.** The wire and props say `labeled_by` — that is the axis: the write DTO field (`labeled_by[]` → `labeledByIds`), `edges.labeled_by` in node detail, validation errors. The PERSISTENCE key stayed `classified_by` — `parsed_meta`, frontmatter on disk, sync wire; renaming it would cost a JSON migration of every node + rewriting the `.md` files in every vault, and it is deferred (see `observations.md`). The boundary is declared in exactly one place — `EdgePredicate::listMirrorKey()`; everything that forwards the mirror outward as is (the row prop `classified_by` = the membership subset, `LabelKind::wireSlug()`) keeps the old name on purpose.

The scalars are needed because sync and frontmatter carry only node fields, not edges; after receipt the edges are reconstructed from the mirrors (`scoped_to` — in a fifth pass, `ScopedToReconciler`, see §7a). **Sync wire:** the `classified_by` uuid is translated into a cross-device-stable **slug** by the decorator `ClassifiedBySlugWireFilter` (the receiver resolves slug→its own node and rebuilds its uuid mirror) — edge = SoT, slug = transport, uuid = local read cache. A cross-device collapse forwards mirror references loser→winner (`ClassifierDedupe::forwardMirrorRefs`, which reads the list of keys from `EdgePredicate` instead of hardcoding it). The cost is several `applyFromScalar` passes.

---
The list mirror is rebuilt as the target uuids in edge creation order; soft-deleted targets are excluded (`EdgeMirror`).

---

## 11b. MCP node control plane (agent surface)

`routes/ai.php` → **`Mcp::web('/mcp/nodes', NodeServer::class)`** under `auth:sanctum` + `CheckAbilities:mcp`. Tokens are self-serve — Settings → **MCP & integrations** (`TokensController`: a personal access token with the `mcp` ability, shown once). The `overy-nodes` server carries an `#[Instructions]` preamble with the protocol conventions: a node is addressed by its 26-character Crockford key (**never invent ids**), dates are ISO-8601 UTC, “what is due today/this week” is `list-nodes` with the `today`/`this-week` view (the user’s tz is applied on the server), there is no delete — archive is reversible, only the acting user’s own nodes are visible.

**11 tools** (`app/Mcp/Tools/Node/`): read — `search-nodes` / `list-nodes` / `get-node` / **`get-attachment`** (returns the attachment file: an image, audio or text; PDF, Word and spreadsheets as text extracted on the server, because a tool answer does not carry documents); write — `create-node` / `update-nodes` / `set-state` / `move-nodes` / `label-nodes` / `archive-nodes` / **`split-node`** (turns a node into a checklist of actionable children).

**There is no batch as a separate concept.** Every write takes `nodes` — one node or fifty; a direction that used to be a second tool became an argument (`remove`, `unarchive`, `type`). There is one answer for all (`Mcp\Support\BatchNodes`): `nodes` — the changed nodes IN FULL, `skipped` — what was skipped and why; a node that the operation did not change appears in neither. Arguments that name the value of ONE node (`title`, `content`, `before`, `after`) refuse a selection.

- **Tools go through the same actions as the web** (`CreateNode`/`UpdateNode`/`MoveNode`/`PlaceNode`/`ReorderMembership`, …), not through controllers — invariants, the scope guard, sync dispatch and layout synchronization come by construction. Owner scope — `OwnedNodeLocator`.
- **Answers use five shared payload shapes** (`App\Mcp\Support`): `NodeSummary` is the compact write echo (key/title/state/priority/dates/archived), `NodeDetail` is built **on top of** it, `NodeListing`/`NodeSearchResults` shape an already-presented row (they were deliberately not folded into `NodeSummary` — that would cost a per-row reload of models the presenter already resolved), `BatchNodes` is the write envelope. Each projector also declares ITS OWN schema (`::schema()`), which goes into the tool’s `outputSchema`, so what is declared and what is returned have no room to drift apart; answers go out as `Response::structured()` — as data, not as a JSON string in text.
- **Batch — one op builder shared with the web** (`Services\Node\NodeBatchOps`): change detection lived inline in controller closures, so the MCP twins did not get it (`batch-meta` ran `UpdateNode` over nodes that already held the value, paying the mirror+push fan-out and reporting the selection size as `applied`). Now a verb that gains a third surface inherits the guard by construction.
- **`set-state` promotes.** Both directions go through `PlaceNode` — the same Command as the web drag-into-queue. The web keeps two gestures (join a group as a member through a state PATCH; drop into a queue through `/place`); the control plane has one state tool, and it means the second — state queues list only actionables, so “put a note into a state” gave an echo of `state: next` and an empty queue.
- **There is no CLI, on purpose.** An Artisan wrapper over the same operations was written and removed: two surfaces over one control plane = a second source of behavior. The CLI promise was removed everywhere the user read it (the docs page, the landing card, Settings).

---
- **`list-nodes` — dimension × container, with a cursor.** `view` picks the dimension (a virtual view, a state slug, a classifier slug or a node key — its children), the optional `container` (a node key) pins it to a subtree: “what is in progress for this project” is one call. The listing is paginated by cursor (`cursor` on input, `nextCursor` in the answer while rows remain; `total` is the size of the whole set). A listing row carries `traits` — an attachment, a note and a tag are distinguishable without `get-node`.
- **The link graph is walked only through `get-node`:** `NodeDetail` returns both sides — `references` (whom the body’s `[[…]]` point to) and `backlinks` (who points to the node; for a classifier it is always empty — its incoming edges are membership). Searching for a token as text gives no backlinks: the brackets are not indexed. Membership (`labels`) does not appear in `references`/`backlinks`.
- **The vocabulary in tool descriptions is generated from code:** the allowed `view` values of `list-nodes` are assembled from `VirtualView::cases()` + `StateVocabulary::line()`, the state slugs in `#[Instructions]` from `StateVocabulary`. There is no hand-made copy of the vocabulary in the descriptions.
