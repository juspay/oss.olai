# Pinning layouts

Brief for an implementing agent. Branch `pinning-layouts`. Read `CLAUDE.md` first: Cordis adherence, docs in the same PR, full e2e coverage, `just ci`.

## The ask

A reader has two panes open — a supervision chat on the left, `orchestrator/lanes.olai` on the right — and wants one click on the sidebar shelf to bring that arrangement back. Today a pin holds one page; `docs/architecture/overview.md` ends its split-panes section with "Saved layouts are a follow-up." This is that follow-up.

## Delivery

One PR, opened as a **draft**, on this branch. The implementor never merges it: run `just ci` until green, mark it ready for review, and stop. A human merges.

## Decisions already made (do not re-litigate)

| Question | Answer |
|---|---|
| Does pinning a layout ask for a name? | **Always.** Same box filter pins use. A layout pin the app writes is always `[Name](/s/…)`. |
| What does ⌘⇧P do? | **Nothing. Remove the chord entirely.** Pinning is the ⌘K row and the row menu only. |
| What does clicking a layout pin do? | **Replaces the whole workspace** with the saved one, as one history push. Back returns to what was there. |
| Does the pin store pane widths and focus? | **No. Pages only.** The stored address is `/s/<pane>/<pane>` with no `?w=` and no `?f=`. Reopens as equal widths, first pane focused. |

## What exists (read these before touching anything)

- **A workspace is already a URL.** `packages/plugins/navigation/src/workspace.ts` — `hrefOfWorkspace`, `workspaceOf`, `WORKSPACE_PREFIX = "/s/"`, `lone`, `isLone`, `panesOf`. The URL is the only place pane state lives; `history.state` holds a key and nothing else (`router.tsx:63-65`).
- **A pin is already a URL in a file.** One node in `_olai/Pins.olai` whose title is an address, optionally wrapped as a markdown link for a written name. `docs/format.md` "## Pins". Wire shape `packages/format/src/shelf.ts:53-103`; browser shape `Pin` in `packages/plugins/pins/src/browser/pins.ts:106-159`.
- **The two don't meet.** `addressIn` (`packages/plugins/navigation/src/address/address.ts:57`) calls `routes.routeIn`, which returns `null` for `/s/…` because no page claims that prefix. So a `/s/…` row in `Pins.olai` is plain text, not a pin. The `pin` verb (`packages/surface/src/edit.ts:835`) carries `at` verbatim and the server does not parse it, so the server half needs no change — verify this by reading the server's `pin` resolver before assuming it.
- **The pin gesture is one function behind three doors.** `packages/plugins/pins/src/browser/Palette.tsx` (`run`), `naming.ts` (`namingFor`, `askName`, `namedEdit`), `pinning.ts` (`togglePin`), `palette.ts` (`pinItem`, the ⌘K row). The chord enters through `packages/web/src/client/keys.ts:122` and `packages/plugins/navigation/src/palette/Palette.tsx:851` → adapter `key("pin")`.
- **Ownership.** `olai-plugin-navigation` owns pane state and the URL grammar, exposed as service `navigation.state` (`Router`, `packages/plugins/navigation/src/routing.tsx:9-82`). `olai-plugin-pins` owns the shelf and consumes navigation through that service. Keep it that way: pins never parses `/s/` itself.

## UX

**⌘K, in a split.** Two rows where today there is one:

```
Pin this page
The orchestrator

Pin this layout…
supervision · orchestrator/lanes.olai
```

- "Pin this page" is unchanged (minus the `⌘⇧P` hint). It still acts on the focused pane's route.
- "Pin this layout…" appears only when `!isLone(workspace)`. The ellipsis is the app's convention for a verb that asks first. Its second line is the pane names joined with ` · `.
- Choosing it asks in the palette: *"a name for this layout — Escape backs out"*. Enter with an empty box is refused with the line *"a layout needs a name"* and the question stays up (unlike a page pin, where empty Enter is the bare pin). Escape writes nothing.
- When the current workspace is already on the shelf, the row reads **Unpin this layout** and is a one-press trash of that row, like page unpin.

**⌘K, on a lone page.** Only "Pin this page". No layout row.

**The shelf.** A layout pin draws with a split mark (two side-by-side rectangles) instead of the pin mark, its written name, and a tooltip listing the pane names. Hover controls (`✎`, `×`) work as on any pin. Rename asks the same question and refuses an empty answer for a layout.

**Clicking a layout pin** replaces the whole workspace (push). Alt/Shift-click on a layout pin does the same thing as a plain click; there is no "open a layout to the right".

**Current-ness.** A layout pin is `current` when the open workspace, stripped to pages, equals the pin's address. A page pin inside that layout is `current` too, as it already is.

**A bare `/s/…` row an agent wrote** (no name) is still a pin. Draw it with the split mark and the pane names joined, deriving each name the way `nameOf` does; a `/#id` pane whose name the browser cannot resolve draws its address. The app itself never writes a bare layout pin.

## Storage

Nothing new in the format. A layout pin is `{"id":"…","ord":"…","title":"[Orchestrating](/s/%23abc/orchestrator%2Flanes.olai)"}`. Extend the Pins section of `docs/format.md` to say a title may be a workspace address: `/s/` followed by percent-encoded page addresses, one per pane; the app writes no `?w=` or `?f=`, and tolerates them when reading.

## Implementation

### 1. Navigation (`packages/plugins/navigation`)

Add to `Routing` (or to `Router`, whichever `Link` and pins can both reach through `navigation.state`):

- `layoutIn(href: string): Workspace | null` — reads a `/s/…` address; `null` for anything else. Tolerates `?w=`/`?f=`/`?a=`/`?t=` when present. Reuses `workspaceOf`.
- `layoutHref(workspace: Workspace): string` — prints a workspace with **fractions and focus stripped** (leaves only, `focus: 0`). Reuses `hrefOfWorkspace` on a normalized copy. Must be a bijection with `layoutIn` on what it writes; add the unit test beside `workspace.test.ts`.
- `open(workspace: Workspace): void` on `Router` — commits the whole workspace as a `push`, landing nothing. Bind it in `router.tsx` beside `openRight`.

Do not add a page claim for `/s/`. `routeIn`/`routeOf` keep returning what they return; a workspace is a level above a route and stays that way.

### 2. Pins (`packages/plugins/pins`)

- `Pin` gets a target: `readonly target: { kind: "page"; route: Route } | { kind: "layout"; workspace: Workspace }`. Keep `route` for page pins if that avoids churn, but `Shelf.tsx`, `Pin.tsx`, `pinnedAt`, `pinsOf` and `Face` must all branch on the kind.
- `pinOf` tries `addressIn` first, then `routes.layoutIn(addressWritten(title))`.
- `Pin.tsx`: for a layout, render an `<a href>` whose click calls `router.open(workspace)`; do not use `<Link>` (it navigates one pane). Split mark, tooltip of pane names.
- `pinnedAt` gains a sibling `pinnedLayout(routes, shelf, workspace)` comparing `layoutHref`.
- `palette.ts`: a second item `pin-layout`, only when `!isLone(nav.workspace())`. Labels: "Pin this layout…" / "Unpin this layout". `place` is the joined pane names.
- `naming.ts`: `Naming` gains `{ kind: "layout"; at: string; panes: string }`. `askingFor` words above. `namedEdit` refuses an empty name for `layout` and for a rename of a layout pin, with the sentence *"a layout needs a name"*. Everything else routes through the existing `pin` verb with `name`.
- `Palette.tsx` (pins): a second `run` for the layout row. Remove the `key:` adapter.
- Remove the `hint: "⌘⇧P"` from the page row.

### 3. Remove the chord

- `packages/web/src/client/keys.ts`: delete the `pin` chord at line 122, its `Action` member, its help entry (~line 672), and the commentary. The file has an invariant test between `CHORDS` and the help table; both change together.
- `packages/plugins/navigation/src/palette/Palette.tsx`: delete the `match.action === "pin"` branch (~line 851) and the doc comment about "two doors" (~line 577).
- `packages/plugins/pins/src/browser/naming.ts`: rewrite the "the press that asked it is dead while it stands" paragraph; the guard itself (`paletteAsking() !== null` in `run`) stays, since the ⌘K row can still be chosen twice.
- Any other `"pin"` key-action plumbing that becomes dead: `pinning.ts`'s `sayPin`/`scopePinSaid` exist for a chord with no panel; check whether anything still calls them after the chord is gone and remove them if not.

### 4. Docs (same PR)

- `docs/format.md` Pins: workspace addresses as titles; app writes named only; no widths.
- `docs/editing.md`: line 31 (chord table row — remove), section "Pinning a page to the sidebar" (lines ~408-442): "three ways on" becomes two; add a subsection "Pinning a layout"; remove every mention of `⌘⇧P` including the "question owns the modal" paragraph at ~426.
- `docs/plugins/pins.md`, `packages/plugins/pins/docs.md`, `docs/plugins/navigation.md`: the new `Router` members and the second pin kind.
- `docs/architecture/overview.md` split panes: replace "Saved layouts are a follow-up." with one sentence on layout pins.
- `docs/architecture/e2e-coverage.md`: the "Panes and sidebar" row.
- `website/`: leave alone unless a picture shows the chord.

### 5. E2E (`packages/tests`)

The chord's removal breaks the `I pin the page` step (`pin_steps.ts`, presses `ControlOrMeta+Shift+p`). Reimplement it through ⌘K: open the palette, type `pin`, choose "Pin this page". Every scenario in `pin_to_sidebar.feature` and the recovery features that used the step must still pass. The scenario at line 124 ("a second press does not wipe the name") is about the chord; replace it with the ⌘K equivalent (choosing the row again while the question is up keeps the box) or fold it into another scenario.

New `pin_layout.feature`, using existing steps from `pane_steps.ts` (`I open the address`, `there are {int} panes`, `pane {int} is showing`, `pane {int} is focused`) and `pin_steps.ts`:

1. Pin a two-pane layout, name it, the shelf shows the name with the split mark, and `_olai/Pins.olai` holds `[Name](/s/…)` with no `?w=`.
2. Resize the panes first; the stored address still has no widths; following the pin reopens equal widths, pane 1 focused.
3. Follow the pin from a lone page: two panes, the right pages. Back returns to the lone page.
4. Follow it from a *different* split: the whole workspace is replaced, not one pane.
5. Empty name is refused and the question stays up; Escape writes nothing; the chord is gone (pressing ⌘⇧P does nothing, shelf unchanged).
6. Unpin from the palette row while in that layout; unpin from the shelf `×`; ⌘Z brings it back.
7. Rename from the shelf; empty rename refused.
8. A pane addresses a node that is trashed: the pin still opens both panes.
9. An agent writes a bare `/s/…` row and a named one into `Pins.olai`; both arrive on the shelf without reload and open.
10. On a lone page, ⌘K offers no layout row.
11. Phone width: following a layout pin lands in the tab strip with the right pages.

Run `just e2e-fast-remote` on the touched features before `just ci`.

## Out of scope

Scroll position (not in any address, by design). Nested or column layouts (codec supports them; the product writes only rows). Opening a layout beside what is open.
