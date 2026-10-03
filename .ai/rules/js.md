---
paths:
  - 'resources/js/**'
  - 'resources/js/**/*.svelte.test.ts'
---

# Js

## A canonical key is an address, never a node id
`canonicalKey` (`node:{key}`, `classifier:{grouper}/{slug}`) is slug-shaped whenever the node has a slug — it addresses a URL, it is not identity. Never feed `split(token).key` into a field that names a node by id (`labeled_by`, `scoped_to`, batch-classify `classifier_id`): the server takes a 26-char Crockford id only and 422s ("labeled_by.N must be a Crockford-Base32 identifier").

Identity comes from the server: the create redirect flashes `createdNodeId` (Crockford) next to `createdNode` (the key). `CreatedMention.id` is the id; `CreatedMention.slug` is the chip token. Two concepts, two fields — a create path that carries only the key is the bug (a tag created from the MetaPanel Labels picker was minted and then attached to nothing).

## Icons render to nothing under jsdom — never assert on an Icon's class
`@iconify/svelte` fetches its icon data, so under jsdom an `<Icon>` renders an empty comment and NOTHING of its `class`, `icon` name or colour reaches the DOM. A test asserting `container.innerHTML` contains `text-destructive` (or any icon class) passes and fails for reasons unrelated to the component.

Assert on something the component itself renders: a `data-*` attribute stating the state (`data-overdue`, `data-channel="telegram"`), an `aria-label`, or a wrapper element's own classes. Split the rule in two when it helps — a lib test for slug → glyph (`fluentIcon.test.ts`), a component test for "the component hands the resolver the right input".
