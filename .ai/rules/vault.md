---
paths:
  - 'resources/js/lib/vault/**'
---

# Vault

## The overlay reconciles against the list buffer, never the card prop
`nodeOverlay` paints an optimistic edit on every surface; a `reconcile*` call takes that paint back off once the server matches. WHICH server value it is fed decides whether the edit survives.

An in-place edit refreshes the open card's `node` prop but deliberately NOT the list's `nodes` buffer (the infinite-scroll window survives — 0c127652). The card is therefore the FIRST surface to catch up and the list buffer the LAST. Reconciling on the card prop drops the override while every list row still holds the pre-edit summary, and the rows snap back to the old value. That is how the icon picked in the node header kept showing for one frame in the node list and then reverting.

Rule: there is exactly ONE reconcile seam for render fields — `reconcileNodeRow(id, row)` — and it takes a `nodes` row (`NodeRowFields`: title/due/scheduled/priority + icon/color/type_icon). It is called only from `NodeList.svelte` (the window) and `Vault.svelte` (the selection sidecar). Do not add a second per-axis reconciler, and never call one from a detail/card component. `nodeOverlay.wiring.test.ts` pins the call sites — the store's own unit tests pass whichever prop a caller feeds it, so only a call-site guard catches the producer going wrong.

State is the same story with an extra step: `holdStateReconcile` pins a state override until the list re-streams.
