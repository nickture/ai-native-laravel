# Instance icon inheritance — design

> **Not implemented as planned.** The plan set up a derived scalar
> `parsed_meta.type_icon` with one writer and a reflow cascade when a type's
> glyph changes. That is what was built, and every new write path had to
> refresh it, until one (inline-create) stopped doing so: typed nodes fell
> back to the default note glyph on reload. The stamp and the cascade were
> removed (collapse-to-read): the glyph is resolved **on read** from the
> `labeled_by`-[type] edge (`Services\Vault\Query\TypeGlyphResolver`), the
> instance stores no `type_icon`, and there is no reflow cascade. The original
> document follows; the authority on the current model is
> `docs/product-architecture.md` §NodeGlyphData.

**Goal.** A classified instance with no explicit icon inherits its **type's** glyph as its default, instead of falling to the plain-note default (`fluent:note-20-regular`). A People-instance (Person) reads as one person; a Project as a folder; etc. The inherited glyph is a **default** — an explicit per-instance icon always overrides it.

## Behavior

Glyph precedence, new step in **bold**:

```
explicit icon → actionable checkbox → state → attachment → classifier-role → INHERITED type icon → note default
```

- **Only classified instances inherit.** A node with a `labeled_by` edge to a `[type]`-trait target borrows that type's glyph. A loose note (no type membership) still falls to `fluent:note-20-regular`. Unchanged.
- **Tags never donate.** Only `labeled_by` targets carrying the `[type]` trait donate an icon. A plain tag/context/area membership (`[classifier]` without `[type]`) does **not** — an instance tagged `#research` but classified by no type stays note-default.
- **Which glyph:** the type's `instanceIcon` if set, else the type's `icon` verbatim.
- **Actionable wins.** `nodeGlyph()` short-circuits on the `actionable` trait before `nodeIcon()`, so a to-do keeps its checkbox regardless of type. The user's "if not actionable" is satisfied structurally — no new gate.
- **Explicit override wins.** A per-instance `parsed_meta.icon` beats the inherited default.
- **Multiple types:** an instance labeled by 2+ `[type]` targets borrows from the first by edge position. Rare; documented + tested.

## The `instanceIcon` field (singular exception)

People is the **only** built-in whose icon is inherently plural (`fluent:people-20-regular`, a group). Every other built-in (folder/calendar/tag/location/target/bot) reads fine on a single instance, so they inherit verbatim.

- SSoT for built-ins: `App\Classifier\BuiltInClassifier` gains `instanceIcon(): ?string` — `People → 'fluent:person-20-regular'`, all others `null` (inherit `icon()` verbatim).
- `MinimalTypeSeeder` writes `parsed_meta.instance_icon` onto the People type node (and any future built-in that returns non-null). Parity-locked by the existing drift test between the enum and the seeder.
- Custom types created by the user get **no icon-picker** in this pass — they inherit their own `icon` verbatim. Cheap to add a picker later if a user hits another plural-glyph custom type.

## Mechanism — derived cache at the classify write-path

The glyph resolver is deliberately own-fields-only (`nodeVisuals.nodeIcon` reads only the node's `{icon, traits, slug}`; every surface forwards the same fields → same glyph everywhere by construction). To keep that invariant and avoid a per-instance type lookup at render (the known vault-read N+1), the inherited icon is resolved once at write-time and stored as a **derived read-cache field** on the instance — the same discipline as the existing `parsed_meta.classified_by[]` uuid mirror (edge = SoT, scalar mirror = read-only cache).

- **New instance field:** `parsed_meta.type_icon` (string | absent). Distinct from `icon` — `icon` stays the explicit user override; `type_icon` is the derived inherited default. Never written by a human.
- **Single writer:** the classifier-membership write path that already maintains the `classified_by[]` mirror — `ClassifiedByCacheWriter` (driven by `ClassifierMembershipDispatcher` / `MentionResolver::sync`). It already loads the classifier target set; extend it to pick the primary `[type]` target and stamp `type_icon = instanceIcon ?? icon`, or clear it when no type membership remains. One writer, no second source.
- **Read:** `NodeGlyphData::from($meta)` reads `$meta['type_icon']` and emits it in `toRow()` / `toCosmetics()`; `nodeVisuals.nodeIcon` gains a `typeIcon` input and the precedence step above. Both mirrors kept in lockstep by the existing `nodeVisuals.parity.test.ts` + `NodeGlyphData` tests.

### Reflow on type-icon edit

The `classified_by[]` mirror stores **uuids** so a type *rename* needs no reflow. Caching a resolved icon string reintroduces a reflow need when a type's `icon`/`instance_icon` changes. Piggyback the existing classifier→instance cascade (`ClassifierRenameCascade` / `RenamePropagator::propagateClassifierSlug`, already fired from `UpdateNode` on classifier/type edits): extend it to re-stamp `type_icon` on the type's instances when the inherited glyph changes. Bounded (one type's instance set), rare, and reuses a trodden path — no new fan-out infra.

## Scope

- **In:** instance → type inheritance (the reported case).
- **Extends edge (sub-type → parent type):** a sub-type created via `CreateSubType`/`extends` inherits the parent's `instance_icon`/`icon` as its own default **type** icon — free if `extends` resolution already loads the parent at create; confirm during implementation and fold in, else defer with a note.
- **Out:** custom-type instance-icon picker; changing any non-People built-in glyph.

## Corner cases (each gets a test)

1. Classified instance, no explicit icon, non-actionable → inherits type glyph (People → person, Projects → folder).
2. Explicit per-instance icon → override wins over `type_icon`.
3. Actionable instance under a type → checkbox, not type glyph.
4. Loose note (no type membership) → note default, `type_icon` absent.
5. Instance with a tag/context membership but no `[type]` → note default (tags don't donate).
6. Un-classify (remove the only type membership) → `type_icon` cleared → back to note default.
7. Multiple `[type]` memberships → first-by-position donates; deterministic.
8. People instance specifically → `fluent:person-20-regular`, not the group glyph.
9. Type-icon reflow: change the type's icon → its instances' rendered glyph updates (via cascade) or is documented-stale — behavior asserted either way.
10. Backend/FE parity: `NodeGlyphData` and `nodeVisuals` resolve the identical glyph for the same payload (parity test).

## Design review (SOLID / GRASP / GoF)

- **SSoT / no second source:** type node owns the glyph; `type_icon` is a derived read-cache with exactly one writer, reconciled through the classifier-membership pass — consistent with `classified_by[]`. No mirror written from two places.
- **Information Expert / Low Coupling:** the write path that already knows the classifier set resolves the icon; glyph surfaces stay dumb forwarders (no new "surface decides role" coupling).
- **Open/Closed:** precedence extended by inserting one ordered step; `BuiltInClassifier` extended with `instanceIcon()` — no branching on specific slugs in resolvers.
- **DRY:** built-in exception lives once in `BuiltInClassifier`, seeded from it, parity-tested; FE/BE glyph logic stays mirrored under existing parity tests.

## Files touched (indicative)

- `app/Classifier/BuiltInClassifier.php` — `instanceIcon()`.
- `database/seeders/MinimalTypeSeeder.php` — seed `instance_icon` for People; drift test parity.
- `app/Services/Outliner/Mentions/ClassifiedByCacheWriter.php` (+ `ClassifierMembershipDispatcher`) — stamp/clear `type_icon`.
- `app/Services/Vault/Presentation/NodeGlyphData.php` — read + emit `type_icon`.
- `resources/js/lib/vault/nodeVisuals.ts` — `typeIcon` input + precedence step; parity test.
- `app/Services/Classifier/ClassifierRenameCascade.php` (or `RenamePropagator`) — reflow `type_icon` on type-glyph edit.
- Tests: `nodeVisuals.test.ts`, `nodeVisuals.parity.test.ts`, `NodeGlyphData` test, classify/reflow feature tests.
