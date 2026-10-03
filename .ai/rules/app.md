---
paths:
  - 'app/**'
---

# App

## labeled_by is the wire name; classified_by is the persist key
The labeling axis is one predicate, `EdgePredicate::LabeledBy` (7). Two names on purpose, split at the persist boundary:

- `labeled_by` — everything transient: the write field (`labeled_by[]` → `UpdateNodeData::LABELED_BY` / `labeledByIds`), `edges.labeled_by` in the node-detail payload, validation error keys, the `/api/states/scoped` query param.
- `classified_by` — the persist key ONLY: `parsed_meta`, frontmatter in `.md`, sync wire. Declared in `EdgePredicate::listMirrorKey()`; read it from there instead of hardcoding a second literal.

Row props that forward the mirror verbatim (`NodeRowAssembler`, `InstanceRowFactory`, suggest/mention rows) and `LabelKind::wireSlug()` keep `classified_by` deliberately — there it means MEMBERSHIP (the mirror subset), not the axis. Renaming the persist key needs a JSON migration + `.md` rewrite; see specs/work/observations.md.

## Hand-setting a node's updated_at means naming touched_in AND touched_by
`nodes.touched_in` is the mutable twin of `born_in`: the channel (BornIn) that made the LAST edit. `nodes.touched_by` is the actor that stood between the owner and that write — an MCP agent named by its token, null everywhere the owner wrote the row themselves — and `born_by` is its immutable twin. Channel and actor are one fact in two columns and always move together. All four are stamped in `Node::updateTimestamps()`, so every ordinary save — including `saveQuietly()`, which skips events but not timestamps — gets them for free.

The stamp deliberately steps aside when the caller sets `updated_at` or `touched_in` itself. So any writer that takes over the row's clock must name the channel AND the actor in the same `forceFill`: `RowChangeStamp`, `FileImporter`, `NodePusher`, `PullApplier`. Add a fifth hand-setter without them and the row keeps a stale one — and a stale ACTOR is worse than a stale channel, because it credits an edit to an agent that had no part in it.

Do not gate the stamp on `updated_at` having MOVED. The column stores to the second, so a second write inside the same second formats to the value already there and Eloquent reports it clean; two MCP calls in one second (create, then label) would keep the first channel and the first agent. The only question is whether the caller took the clock, which is answered BEFORE the parent moves anything.

The value comes from `App\Support\WriteChannel` (scoped) — `BornIn::runtime()` and no actor, unless an entry point declared otherwise. `declare()` takes the pair and defaults the actor to none, so a channel can never be set while a stale actor survives beside it: `NodeServer` declares `(Mcp, token name)` in its constructor (covers HTTP and the test harness), `FileSyncService::apply` wraps each watcher event in `during(Filesystem)`. Sync carries both over the wire (`WireProvenance` parses and caps them at the trust boundary) and applies the same LWW verdict `updated_at` gets — a losing push must not repaint either. A peer that names the channel but no actor UNSETS the local one: it has described this edit, and inheriting the previous agent's name would sign a new edit with an agent that had no part in it. Silence about both leaves both alone.

Tests that make several requests must put the request boundary back by hand (`forgetGuards()` + `forgetScopedInstances()`, what Octane does per request). One container across requests means the guard keeps the FIRST call's token and the channel keeps what it declared — a browser edit then records as the agent's.
