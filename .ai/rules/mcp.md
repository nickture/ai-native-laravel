---
paths:
  - 'app/Mcp/**'
---

# Mcp

## The MCP reference graph is walked through get-node, never through search
`get-node` returns `references` (outgoing) and `backlinks` (incoming), both projected from `Services\Graph\Backlinks` — the same service behind the MetaPanel, so the agent view and the UI cannot drift. Never resolve a `[[…]]` token by full-text searching for it: the brackets are not indexed and the hits are unrelated.

A classifier reports no backlinks by design — its incoming `labeled_by` edges are memberships, not references. That direction is `list-nodes` with the classifier's slug.

A filter the parser cannot honour (`type:` / `trait:` values are ASCII slugs) must be REFUSED at the tool, not folded into free text. The fold stays correct in `FilterParser` for mid-keystroke Spotlight — the machine boundary consults `rejectedFilters()`.

## The control plane emits Crockford keys, never the vault's canonicalKey
`canonicalKey` is a ROUTING token (`node:{crockford}`, `classifier:{grouper}/{slug}`, `view:today`) that the FE splits into a Wayfinder route. Forwarded to an agent it is a dead end — a classifier key names no node any tool accepts, and prefixed keys get refused by the very tool the row exists to be passed to. Every row id goes through `Mcp\Support\RowKey`: a node presents its Crockford key, a non-node presents the naming half of its token. `OwnedNodeLocator` still strips a `node:` prefix, for keys an agent cached before this.

## A cross-cut list goes through LoadVaultPage's scope, never a second filter
`list-nodes` takes `container` (a node key) alongside `view`: `view` is the dimension, `container` pins it to that node's content set. Both arrive at `LoadVaultPage::handle($ownerId, $view, null, $scope)` — the SAME call the web's Summary drill-down makes, so the tool and the UI cannot answer differently. `RenderScopedList` is not the owner of that logic; it only maps (dimKind, dimKey) to a view key.

Never add a filter param to a tool that the vault expresses as a view. `state`, `priority`, `label` and the date windows are all dimensions already: a state slug, `prioritized`, a classifier slug, `overdue` / `today` / `this-week` / `aged`. A parallel filter language over them is a second answer to "what is in this slice".

Parity is locked in tests/Feature/Mcp/ListNodesIntersectionTest.php — the tool's rows and the `vault.scoped` route's rows are compared by id and order.

## A state argument is a string checked per owner, never a JSON-Schema enum
The state vocabulary is open: a state is any node with the `state` trait, and `StateSlug` names only the seven a fresh vault seeds. Publishing `->enum(StateSlug::slugs())` tells an agent the states the user made do not exist, and `Rule::in(StateSlug::slugs())` then refuses them ("The selected state is invalid").

`set-state` and `create-node` take a plain string and ask `Mcp\Support\StateVocabulary::knows($ownerId, $slug)`, which reads `StateMachine::validSlugs` — the same list `PlaceNode` validates against on the web, so the tool refuses exactly what the domain would. Per owner, because a stranger's "backlog" must not resolve. Unknown slug answers `Unknown state 'x'.`, matching the `Unknown view` shape.

`StateVocabulary::line()` is the one place naming the canon plus its open half; `list-nodes`, `set-state` and `create-node` all read it. The same reasoning applies to any other open vocabulary (classifiers, custom views): closed enum in the schema = a surface the agent never tries.
