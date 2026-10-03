---
paths:
  - "resources/css/**"
  - "resources/js/**/*.svelte"
  - "resources/views/**"
---

# Overy — Design

> Overy’s design system: decisions, exceptions and token names. Token values live in `resources/css/app.css` and are not repeated here. The rules of the `nickture-interface` skill apply unless this file says otherwise. Vision and principles — `specs/foundation/project.md`, copy — `specs/foundation/text.md`. Where code diverges from this file, the entry lives in `specs/work/observations.md`, section “Code diverges from Foundation”.

## 1. Brand and interface principles

- **The mark is O_:** a dark “o” and an underscore in the `--brand` color. The underscore is orange on every medium, including the desktop icon. The dev build differs by a yellow tile (`public/icons/icon-dev.svg`).
- **Asset sources.** The mark in the interface is `OveryMark.svelte` (the “o” in `currentColor`, the underscore in `fill-brand`). Files with a fixed address (favicon, manifest, PWA icons, `og.png`) live in `public/`. Icon sources for all platforms live in `resources/icons-source/`. Fonts and images load through Vite from `resources/fonts/` and `resources/images/`, so the file name carries a hash.
- **One visual device, one meaning.** Strikethrough means only “done”, `opacity-50` means deleted or a ghost, a tint means a drag target. A new device is first checked against the ones already taken.
- **A new action fits into an existing surface:** a MetaPanel section, QuickCapture, a picker. A separate dialog appears only when the flow is tied to no surface.
- **A picker’s action is the first item of its list**, not a button next to the list.
- **A frequent action has a key and a visible `Kbd` hint.**

## 2. Character

| Axis | Overy |
|---|---|
| Airy ↔ dense | Dense in navigation, the sidebar and lists, airy in the document body |
| Own form for each type ↔ difference by layout | Difference by layout: “everything is a type”, types differ by glyph and label, all nodes share one layout |
| Cohesion ↔ composability | Composability: graph, labels, types and scoped views instead of ready-made scenarios |
| Conventional ↔ surprising | Conventional: Notion and Obsidian patterns, standard keys |
| Literal ↔ metaphorical | Literal. The only metaphor is the mountain glyph of Top |
| Evolution ↔ revolution | Evolution |

## 3. Recognizability

Overy is recognized by the features of its system, not by a picture: the O_ mark and the orange `--brand` as the only brand accent, the Overused Grotesk typeface, the NodeGlyph checkbox on every node, chips with node titles, one Fluent icon system, visible key hints.

## 4. Accessibility

The minimum is WCAG 2.2 AA. The accessibility rules of `nickture-interface` apply without exceptions. Token contrast measurements are in section 5.

## 5. Color

| Role | Token |
|---|---|
| Page background | `background` |
| Text | `foreground` |
| Muted text | `muted-foreground` |
| Muted surface (code, empty areas) | `muted` |
| Card and popover | `card`, `popover` |
| Hover and selection on a white surface | `accent` (= `neutral-150`) |
| Control on a card surface | `secondary` |
| Primary action | `primary` |
| Brand: the mark and rare accents, not an action color | `brand` |
| Error and deletion | `destructive` |
| Border | `border` (semi-transparent, reads on white and on `neutral-150`) |
| Input field border | `input` |
| Focus | `ring` |
| Search match | `highlight` |
| Sidebar | `sidebar-*` |
| Syntax highlighting | `--code-*`, only in `editor.css` |
| Intermediate gray step | `neutral-150` |
| Success, warning, info | No tokens. Decided: green, amber, blue, as in `SyncIndicator` today |
| Disabled | `opacity-50` on the control, no separate color |

- Markup uses only semantic utilities. Raw palette colors, hex and `dark:` pairs are not written in components. If a token does not fit a surface, its definition in `app.css` changes. An intermediate step gets a token.
- **Exceptions:** the user node palette (`colorCatalog.ts`) and priority colors (`priority.ts`) are data, not roles. One more exception is the graph canvas.
- **The dark theme** exists only inside the vault: the `.dark` class on `<html>`, the user picks system, light or dark. Marketing (`.landing`: landing, changelog) is always light through `html:has(.landing)`; sign-in pages are light because `AuthSimpleLayout` removes `.dark` while they show. `--brand` is the same in both themes.
- **Contrast** (measured on current values). Pass: `foreground`, `muted-foreground` on `background`, `primary`, `ring`, sidebar, `highlight`. Fail AA:

| Pair | Light | Dark | Required |
|---|---|---|---|
| `muted-foreground` on `muted` / `accent` / `secondary` | 4.35 / 4.05 / 3.76 | ok | 4.5 |
| `muted-foreground/60` on `background` | 2.30 | 3.44 | 4.5 |
| `destructive` as text on `background` | 3.76 | ok | 4.5 |
| `destructive-foreground` on `destructive` | 3.60 | 3.62 | 4.5 |
| `brand` on `background` (text and icon) | 2.89 | ok | 4.5 / 3 |
| `input` on `background` (field border) | 1.26 | 1.31 | 3 |
| `sidebar-primary-foreground` on `sidebar-primary` | — | 1.00 | 4.5 |

## 6. Layout

- The step is 4px (Tailwind `--spacing`). Spacing and sizes come from the scale.
- Breakpoints are the Tailwind defaults (`sm` 640, `md` 768, `lg` 1024, `xl` 1280), and only for the page frame: sidebar, columns, header. JS does not duplicate a breakpoint as a number.
- A component inside a panel responds to the width of its container through `@container`, not to the window width.
- The app is a sidebar and a work area. The landing content width is `max-w-360`.
- Utilities are written in canonical form (`max-w-360`, `underline-offset-6`). Square brackets only when there is no canonical form: `em` and `lh` on elements that grow with the font size, percentages, compound grids. `h-[1lh]` does not become `h-1lh`: that utility does not exist.
- `flex-1` with inner scrolling gets `min-h-0`.
- Marketing on iOS: horizontal overflow is suppressed with `overflow-x: clip` on `.landing`, never on `html` and `body`, otherwise `position: fixed` breaks in iOS Safari. The fixed header is opaque and `theme-color` is white, so content does not show through under the status bar.

## 7. Typography

- One typeface — Overused Grotesk: variable, 300–900, lives in `resources/fonts/`, `font-display: swap`. The fallback is the system stack, mono is the system `ui-monospace`. No fonts load from the network.
- Body text is 16px (`text-base`): in content, in dropdown menus, pickers and calendars. Menu items inherit the base size and get no explicit `text-sm`. An item built on `Button` gets `text-base` explicitly: the base `Button.svelte` sets `text-sm`.
- `text-sm` — secondary metadata and counters. `text-xs` — badges only. Arbitrary sizes (`text-[11px]`) are not used.
- Display sizes `text-display-{hero,xl,lg,md}` — marketing only.
- In the editor, headings follow an `em` scale from the base size (`editor.css`), paragraph rhythm is `1lh`.
- Numbers in columns and counters use `tabular-nums`.

## 8. Elevation and layers

- Inside the app, surfaces are separated by background and border, without shadow. Only floating surfaces have a shadow, through `surface-panel`. The modal backdrop is `surface-scrim`. Preview cards in the editor (link, file) are set apart by a thin `border-border` border.
- The layer scale from bottom to top: base, raised (sticky inside a panel), header, overlay (popover, menu, dialog), stacked dialog (a dialog over a dialog), toast, loader. The values get tokens. Arbitrary `z-[…]` are not used.

## 9. Radii

Radii derive from `--radius` through `rounded-sm/md/lg`. Control, chip and tab — `rounded-md`, dropdown menu and popover — `rounded-lg`, modal — `rounded-2xl`, pill and avatar — `rounded-full`. Bare `rounded` is not used.

## 10. Motion

- Durations: press and tooltip — 150ms, menu and popover — 200ms, modal and sheet — 300ms. 300ms is the ceiling for interface response. The exception is marketing choreography (landing, demo).
- Curves: enter — `cubic-bezier(0.23, 1, 0.32, 1)`, move — `cubic-bezier(0.77, 0, 0.175, 1)`, sheet — `cubic-bezier(0.32, 0.72, 0, 1)`. Durations and curves get tokens.
- Keyboard actions and frequent actions (tabs, menus, hover on working buttons) are not animated.
- Reduced motion works in levels: movement is removed, responses become fades shorter than 200ms, loading indicators stay.
- View transitions between pages are off on touch devices.

## 11. Icons

- One system — Fluent through `@iconify/svelte`, the `-20-regular` variant, the active state is `-20-filled`. Third-party service logos — Phosphor `ph:*-logo-light` (`-fill` on hover): Fluent has no brand marks. Emoji and a text glyph only when the user chose them: an icon without `:` renders as text.
- The default size is `size-5` (Fluent’s native 20px). `size-4` only in compact controls: `sm` buttons, items with `text-sm`. In a chip, the icon size is the same at every call site.
- Names by meaning go through the maps in `resources/js/lib/fluentIcon.ts` (states, classifiers, views, UI). A node’s leading glyph is drawn only through `NodeGlyph.svelte` and `nodeGlyph()` from `nodeVisuals.ts`.
- Reserved: Top — `mountain-trail`, Home — `home`, Locate — `location-arrow`, scoped — `location-target-square`, graph — `molecule`.
- No stroke: Fluent uses fill.

## 12. Browsers and systems

- Desktop is the embedded Electron (Chromium). Everything it supports is allowed.
- Web — the last two major versions of Safari, iOS Safari, Chrome, Edge and Firefox. The base is Baseline Widely Available. Features beyond it (`field-sizing`, view transitions, `@starting-style`) are progressive enhancement: the interface works without them.
- Mobile app (Phase 7) — decided when the phase starts.

## 13. State sandbox

The `/dev/states` route is available only in `local` and `testing` and is closed to indexing. It replaces the `/mocks/n` stub. Columns are states (default, hover, focus, pressed, disabled, loading, error), rows are sizes and variants. The page has every primitive from `components/ui/` and every wrapper over it.

## 14. Tokens before components

A component takes values only from `app.css` tokens. If a token is missing, it is first added in `@theme` (for a color, as a pair in `:root` and `.dark`), then used in the component. The exceptions are the same as in section 5.

## 15. Components

- `resources/js/components/ui/` is vendored shadcn-svelte code (`new-york-v4` on bits-ui). It is not edited in place; it is updated through the CLI.
- Custom styling and behavior are built as wrappers outside `ui/`.
- An existing component is used first. A new primitive appears only when nothing ready exists. The reason is recorded here, and the primitive itself lives outside `ui/`.
- A repeated element is one component and one state store by id, with no copies of markup per mode.
- Custom primitives:
  - `chip`: labels, metadata and values. There is one geometry, in `em`, in `resources/js/lib/vault/chipGeometry.ts`: both `Chip` (MetaPanel pills) and the inline `MentionDisplay` take it, so the chip scales with the text of the row, the body and the heading. At a 16px font size it gives `h-7`, `px-1.5`, `gap-0.5` and a `size-5` glyph. A chip inside another chip’s title is drawn with the same recipe at any depth; only the glyph differs — one step smaller.
  - `segmented-tabs`: the tab pill repeats the chip geometry at 16px (`h-7`, `px-1.5`, `rounded-md`). Node tabs are quiet: `border-0 shadow-none`.
  - `section-header`: a collapsible section header, currently only in the type template editor.
