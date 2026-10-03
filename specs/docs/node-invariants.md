# Node invariants — reference matrix

SoT for invariants over `nodes` (traits + scalar properties + edges). The concrete rules are enforced in code; this file is an operational reference and does not duplicate the code.

## Dimensions

**Traits** (`parsed_meta.traits[]`): `classifier`, `state`, `type`, `actionable`, `attachment` (`contact`/`agent` are retired — MVP scope; `attachment` is auto-assigned).

**Scalar state-bearing** (in `parsed_meta`):
- `state` (slug → `now_in` edge; **never stored** — extract → transition → discard)
- `scheduled_at`, `due_at`, `priority`
- `template.{apply_traits, properties, extends, abstract, slug}`
- `protected` (bool, identity flag) — non-deletable/non-archivable (see below)
- `aliases[]` — search-only tokens (outside the display/routing scalars)

**Edge mirrors** (in `parsed_meta`; all values are **UUIDs**, there is exactly one writer — `EdgeAttachmentService`, see below):

| Key | Edge | Shape |
|------|-------|-------|
| `parent_id` | `child_of` | scalar (`scalarMirrorKey`) |
| `now_in` | `now_in` | scalar; `prev_in` is the undo key, also a uuid |
| `extends` | `extends` | scalar (was the parent’s slug — moved to uuid; `ClassifierRenameCascade` died together with its cause) |
| `classified_by[]` | `labeled_by` (the **membership** subset) ∪ manual plain labels | ordered list (`listMirrorKey`) |
| `scoped_to[]` | `scoped_to` | ordered list |

Member order lives in separate scalars (`state_position`, `classifier_positions[slug]`), not in `edges.position`: for `labeled_by` the member rank collapsed to creation order; for `now_in` the write-only position channel was removed.

**Terminology (important: the keys lag behind the model).** The predicates `classified_by`(3) and `mentions`(5) are **retired** — both axes collapsed into a single `labeled_by`(7), and membership vs reference is decided by the nature of the target (`LabelKindResolver`), not by the predicate. But the on-disk/wire key of the mirror kept the old name — `parsed_meta.classified_by`; renaming it would cost compatibility with frontmatter already written. Read it as “the mirror of labeled_by edges”, not as a live predicate. UI copy does **not** use the words classify/classifier — it says “label”/“labels” everywhere (hotkey L).

**Column-level**:
- `nodes.archived_at` — nullable timestamp. NOT a trait. The visibility filter excludes it by default; the sidebar `Archive` view = `whereNotNull('archived_at')`.

**Slug** — auto-derived from the title for classifier-traited nodes.

## Mental model

`classifier` = base trait. `state` and `type` are specializations of a classifier. Toggling `state`/`type` silently bundles `classifier`. Removing `classifier` is blocked while `state` or `type` is present. A pure classifier (tag, context) has only the `classifier` trait, without state/type.

**Classifier-side vs instance-role**: traits split into two roles:
- **Classifier-side** (definition): `classifier`, `state`, `type` — the node is a definition.
- **Instance-role** (member): `actionable`, `attachment` — the node is a member of the graph.

Definition and instance-role are mutually exclusive: one node cannot be both a definition and an instance. **6 forbidden combos** (3×2) are rejected on the BE and auto-stripped in the FE toggle (`toggleTraitLastWins` — last wins: adding a trait strips the conflicting one, so neither QuickCapture nor MetaPanel can assemble an invalid combo). The nature of a node is losslessly reversible along the ladder note ⇄ `[classifier]` ⇄ `[classifier,type]` — the toggle does NOT touch `labeled_by` edges.

## Invariant table

| # | Rule | BE enforcement | FE enforcement |
|---|------|----------------|----------------|
| 1 | `state` ⊕ `type` mutex | `NodeInvariantValidator::assertValid` | `MetaPanel.svelte` MUTUALLY_EXCLUSIVE + `mergeTraits.ts` |
| 2 | `state` ∈ traits ⇒ `classifier` ∈ traits | `NodeInvariantValidator` | `MetaPanel.svelte` toggle bundling |
| 3 | `type` ∈ traits ⇒ `classifier` ∈ traits | `NodeInvariantValidator` | `MetaPanel.svelte` toggle bundling |
| 4 | `classifier` removal blocked if `state` or `type` is present | `NodeInvariantValidator` (via 2/3) | `MetaPanel.svelte` disable + tooltip |
| 4a | classifier-side ⊕ instance-role (3×2) | `NodeInvariantValidator::classifierAndInstanceRole` | `mergeTraits.ts::toggleTraitLastWins` auto-strip |
| 5 | `parsed_meta.state` ≠ null ⇒ `actionable` ∈ traits | `NodeInvariantValidator` | `MetaPanel.svelte::keepsState` guard |
| 6 | `scheduled_at` ≠ null ⇒ `actionable` ∈ traits | `NodeInvariantValidator` | `MetaPanelTiming.svelte` hide if not actionable |
| 7 | `due_at` ≠ null ⇒ `actionable` ∈ traits | `NodeInvariantValidator` | `MetaPanelTiming.svelte` |
| 8 | `template.*` ≠ ∅ ⇒ `classifier` ∈ traits | `NodeInvariantValidator` | `MetaPanelClassifierExtras` triggers on classifier |
| 9 | `labeled_by` — **no** target-trait restriction; membership vs reference is derived from the nature of the target | `LabelKindResolver::of` (not `TargetTraitInvariant`) | label-picker filter `without=state,type`; identity-picker `kind=type` |
| 9a | `[state]` target of `labeled_by` ⇒ `PlainReference` (state = mention only, never classifies) | `LabelKindResolver::of` (checked before the classifier test) | states are outside the `#` picker |
| 10 | `now_in` target carries `state` | `TargetTraitInvariant` | State picker filtered to state-traited |
| 11 | `extends` target carries `type` | `TargetTraitInvariant` | Types-only autocomplete |
| 12 | `child_of` — no trait restriction | n/a (allow any) | allow any |
| 12a | classifier-side source ⊄ `now_in` state (ortho) | `SourceTraitInvariant` | State picker hidden for classifier-side |
| 12b | classifier-side source ⊄ `labeled_by → [state]` (ortho, membership facet) | `SourceTraitInvariant` | Classify autocomplete filters |
| 13 | No self-edge (any predicate) | `SelfEdgeInvariant` | Classify autocomplete excludes self |
| 14 | `child_of` acyclic | `CycleInvariant` | (rely on BE) |
| 15 | `extends` acyclic | `CycleInvariant` | (rely on BE) |
| 16 | `labeled_by` membership subset acyclic | `LabeledByCycleInvariant` (per-target; plain-reference early return) | (rely on BE) |
| 17 | `now_in` cardinality = 1 | `EdgePredicate::cardinality()` = `Cardinality::Single` + actions replace | single-select dropdown |
| 17a | `child_of` cardinality = 1 (single outline-parent) | `Cardinality::Single` + `replaceSingleOutgoing` | reparent displaces prior |
| 17b | `extends` cardinality = 1 | `Cardinality::Single` | single-select |
| 18 | Classifier-traited node ⇒ slug auto-derived from title | `Node::ensureClassifierSlug` boot hook | n/a |
| 19 | Every edge mirror has **exactly one** writer — the edge write seam | `EdgeAttachmentService` (`attach`/`detach`/`detachAllOutgoing` → `EdgeMirror`/`ListMirrorWriter` through `StampedMetaWriter`). Exception — a manual plain label: `NodeEdgeApplicator` writes its subset, the rebuild merges them | n/a (the FE does not write mirrors) |

## File routing (file path allocator, not an invariant but related)

| Trait combo | Shard |
|-------------|-------|
| `state` | `_states/` |
| `type` (without classifier-as-state) | `_types/` |
| `classifier` under a classifier parent | `_classifiers/<parent-slug>/` |
| otherwise | date shard |

See `FilePathAllocator::reservedShardFor` + `scopeClassifierSlug`.

## Scope availability (write-seam guard, not a trait/edge invariant)

`ScopeInvariant` (`app/Services/Graph/Invariants/ScopeInvariant.php`) is not in the trait/edge matrix above; it is a **policy on top of the graph**: a scoped classifier (a tag/state with `scoped_to`) can be attached or entered only inside its `scoped_to` target + descendants. The core is `ClassifierScopeResolver` + `GroupScopeResolver` (one scope SoT). It gates **only user write seams**, **delta-only** (newly added memberships; re-saving a legacy out-of-scope node does not return 422), and checks scope against the resulting placement:

| Axis | Seam |
|-----|------|
| Tags (`classified_by`) | `NodeEdgeApplicator::applyClassifiedBy` (Classify / Update `labeledByIds` / Create / batch) |
| States (`now_in`) | `PlaceNode` (place/placeById) + `UpdateNode` (transitionState + inline slug-command) |

Deliberately **NOT** in `StateMachine::transition` — that method is shared with the sync-wire path, which replicates the canon unguarded (LWW). Rejection → 422 (`ValidationException`). Owner isolation is held separately on each seam; this closes the scope-**within**-owner hole.

## Protected nodes (lifecycle policy, not a trait/edge invariant)

`Services\Node\NodeProtection::isProtected()` — the durable flag `parsed_meta.protected` (stamped once at seed time, **identity, NOT slug** — survives a rename of title+slug; syncs with the node, global across devices). A protected node **cannot be deleted or archived** (the state machine must not break). SoT for the list of protected state leaves — `StateSlug::isSystemCritical()`.

| Protected (load-bearing) | Disposable |
|---|---|
| States-grouper, Inbox, Next, Done, Cancelled | In progress, Waiting, Someday + the whole user vocabulary |

Enforcement: `DeleteNode::handle` + `UpdateNode::handle` (archive branch) → `ValidationException` (single 422 / batch skip-and-report). The sync `DeleteController` is **not** gated (replication must apply). `NodeDetailPresenter.protected` → MetaPanel hides Archive+Delete. The seeder stamps on new installs, a forward migration backfills existing ones. Contract — `SEED_PROTECTION_CONTRACT` (whole-map + per-seed survival canary).

## Validator entry points

Cross-property invariants are checked at:
- `app/Actions/Node/CreateNode.php`
- `app/Actions/Node/UpdateNode.php`
- `app/Services/Sync/PullApplier::applyIncoming`

Edge invariants — `EdgeAttachmentService` (chain `SelfEdgeInvariant` → `TargetTraitInvariant` → `RequiredSourceTraitInvariant` → `SourceTraitInvariant` → `CycleInvariant` → `LabeledByCycleInvariant`).

## Failure mode

Strict reject — invalid combos → `InvalidNodeInvariantException` (422 API). Loud failures policy; silent auto-cleanup was rejected.
