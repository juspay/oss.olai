# Tabs: implementation plan

The approved look is `mockups/tabbed-ui.html` in this directory. Open it in a browser
before reading further; every behaviour below was first shown there. This document is
the plan an implementing agent follows. It names owners, lifetimes, files, tests and
docs, in the order to build them. It is not app code and is not a user doc.

## Decisions already taken

These were put to the maintainer and answered. Do not reopen them.

| Question | Decision |
| --- | --- |
| Where does the set of open tabs live? | Per browser, like sidebar width. The address bar shows only the tab in front. |
| Browser back and forward | Per tab. Back moves within the tab in front. Closing a tab drops its history. |
| Who owns the tab list? | A new plugin, `packages/plugins/tabs`. |
| A chat starts needing you | Its tab wears the dot. Nothing steals focus, no doorbell beyond what exists. |
| Phones | No tabs below the desktop breakpoint. The existing pane strip stays. The stored set is kept but not drawn or driven. |

## Vocabulary

- **Tab**: one entry in an ordered list. It holds a *workspace* (the navigation plugin's
  `Workspace`: a `Leaf` or a `Split` with a focus), so a tab may hold a whole split. A tab
  also holds a title snapshot and the key of its current history entry.
- **Front tab**: the one whose workspace the router is drawing. The address bar is its
  address. Exactly one tab is in front whenever the plugin is active.
- **Strip**: the row of tabs drawn above the panes in the main column, desktop only.
- **Lane**: the navigation plugin's name for "which tab a history entry belongs to". The
  tabs plugin never touches `history`; it names a lane and the router does the rest.
- **Door**: a link in the sidebar or elsewhere that navigates. Unchanged by this work.

## Ownership at a glance

| Thing | Owner | Lifetime |
| --- | --- | --- |
| The tab list, the front tab, persistence, the strip, keyboard chords, palette commands, the link context menu | `tabs` plugin | The tabs row's activation scope |
| History lanes: stamping entries, seeking on popstate, forgetting a lane | `navigation` plugin | The navigation row's activation scope, as today |
| The seat above the panes and the `--height-strip` variable | `layout` plugin | The layout row's activation scope |
| "Open a pinned layout as a tab" | `pins` plugin, in an optional component gated on `tabs.state` | That component's scope |
| Which tab wears the needs-you dot | `tabs` plugin, in an optional component gated on `chat.state` | That component's scope |

Nothing here is a singleton in a utility package. Every live value crosses a package
boundary as a declared service or a contribution to a declared location. Pure helpers
(list operations, the persistence codec, the seek decision) are static and tested alone.

## Behaviour specification

Each row is a requirement and has at least one e2e scenario in the test section.

### The list

- The plugin activates with the stored set if there is one, else one tab holding the
  workspace the address bar already shows. The front tab always mirrors
  `router.workspace()`. When the router moves (a door, back, a pane verb), the front
  tab's record updates. Background tabs never change while in the background.
- `open(workspace, { behind })` appends a tab after the front tab. With `behind` the
  front tab stays in front. Without it the new tab comes to the front.
- `show(id)` brings a tab to the front. This is not a history event.
- `close(id)` removes a tab. If it was in front, the tab to its right comes to the front,
  else the one to its left. Closing the last tab leaves one tab on the front page
  (`HOME_ROUTE` from `olai-plugin-navigation/routes`, the same one the rail's home
  button uses) so the main column is never empty.
- `closeOthers(id)`, `duplicate(id)` (a copy placed right after, brought to front),
  `reorder(from, to)`.
- Below the desktop breakpoint every verb still works on the list, but `open` also
  brings the new tab to the front so a caller needs no branch, and nothing is drawn.
  The stored set is still written, so a phone does not erase a desk.

### The strip

- Drawn in the layout's new seat above the panes, only when `layout.shell.desktop()`
  is true. Height is one token, `--height-tabs: 2.625rem`, published by layout as
  `--height-strip` when a seat is filled and `0px` otherwise, so the pane sheet's
  height budget subtracts it.
- Paint follows the mockup: desk ground, one-pixel rule under it, the front tab on paper
  with the rule broken under it, others muted. A tab is `role="tab"` inside
  `role="tablist"`, with `aria-selected`, a kind glyph, a title, an optional needs-you
  dot, and a close button that appears on hover and on the front tab. The front tab is
  scrolled into view when it changes.
- Right-click on a tab opens a menu with Duplicate tab, Close other tabs, a hairline,
  Close. Use the same menu primitives the outlines ••• menu uses (`MENU_PANEL`,
  `MENU_ITEM` in `packages/ui-primitives/src/menu.ts`), and Kobalte's context menu if
  the existing dropdown wrapper does not already cover it.
- Middle-click on a tab closes it. Drag reorders, using the same threshold and pointer
  helper the pane header uses: `drag` from `@olai/web/client/pointer.ts`, threshold 8.
  It is already a shared package, so nothing needs a new door.
- A `+` at the end opens a front-page tab. A mono readout at the right shows the front
  tab's address. Below the breakpoint none of this exists in the DOM.

### Titles

A tab in the background has no page mounted, so its title is a snapshot taken while it
was in front: `navigation.info(focus).title` when a page reported one, else the route
label. The label is `layout`'s `labelOf(route)` today; move that function into the
navigation plugin as `Routing.label(route)` so both layout and tabs read it through the
`navigation.state` service, and make layout's `pane/label.ts` a one-line delegate. A
split tab's title is its leaves' labels joined with " + ". Store the snapshot with the
tab so it survives a reload.

### Opening things in a new tab

- Any in-app link (an anchor whose href parses through `router.routes`) gets a right-click
  menu with Open and Open in new tab. The tabs plugin installs one document-level
  `contextmenu` listener inside its scope and removes it on release. It ignores anchors
  inside a pane's own content that already own a context menu (the outlines ••• menu
  marks its rows; check for an ancestor with `data-menu-owner`, and add that attribute in
  outlines if it does not exist yet).
- No palette rows in this round. `app.palette` entries carry an `href` only and
  `app.command` is a typed-line verb behind a prefix, so neither fits "close the front
  tab". A palette row that runs a verb would widen `AppPalette`; it is listed under
  later work.
- A pinned layout on the shelf opens as a new tab when the tabs plugin is active, in place
  otherwise. See the pins section.
- Deviation from the mockup, and why: the mockup made ⌘/Ctrl-click and middle-click on a
  door open an olai tab. In the app those gestures already open a *browser* tab, and
  `followLayout` deliberately steps aside for them. Keep the browser's meaning. If the
  maintainer wants a modifier for olai tabs, add it later as a chord on `app.keys`; do
  not take the browser's gesture. The hover ⧉ affordance on sidebar doors is also
  deferred: each door is drawn by its own plugin, so it needs a per-plugin change rather
  than one owner. The context menu covers the same need with one owner.

### Keyboard

Register through the `app.keys` slot. Its contract (`AppChord` in
`packages/plugins/navigation/src/slots.ts`) is one letter, an optional Shift, a
`whileEditing` flag, the words for the shortcuts sheet, and `press`. The modifier is not
the plugin's to choose: every chord in this app is ⌘ on Apple and Ctrl elsewhere, decided
once in `packages/web/src/client/keys.ts`, and `matchKey` rejects Alt. So the mockup's
Alt+digit chords cannot exist here, and the browser owns ⌘T, ⌘W, ⌘1 to ⌘9 and ⌘⇧[ ⌘⇧]
for its own tabs, so those cannot be received either.

Two facts about the slot the implementer must act on:

- Nothing dispatches registered `app.keys` chords today. The palette's key handler in
  `Palette.tsx` matches only the core `CHORDS` table, and the shortcuts sheet lists only
  those. Step 1 below makes the palette component read `hung("app.keys")` through the
  `Faces` service it already holds, match them after the core table with the same
  modifier and exact-Shift rules, refuse a chord whose letter and Shift the core table
  already answers (a console grumble naming both, as `commandsIn` does for prefixes),
  and list them in the shortcuts sheet with `said`. The core table stays in `@olai/web`.
- Chords are letters, so tab verbs get letters with Shift, chosen against the core table
  (`k`, `\`, `j`, `z`, `⇧z`, `⇧w`, `o`) and the browser's reserved set. Proposed:

| Chord | Verb |
| --- | --- |
| ⌘⇧. and ⌘⇧, (Ctrl elsewhere) | Next and previous tab |
| ⌘⇧O | New tab on the front page |
| ⌘⇧X | Close the front tab |

If a proposed letter turns out to be taken on one platform, pick another and record it
in the plugin's docs; do not take a browser chord. All are inert below the breakpoint,
and all say `whileEditing: true` since they are about the page, not the caret.

### Per-tab history, in the navigation plugin

The browser has one history stack. Per-tab back means the router must skip entries that
belong to other tabs. The router already stamps each entry with a key it minted; this
extends the stamp and the popstate handler. Nothing changes when no lane is in force, so
the navigation plugin keeps working without the tabs plugin.

Entry shape becomes:

```ts
interface Entry {
  readonly key: string        // as today
  readonly lane: string | null // the tab this entry belongs to; null = no owner
  readonly at: number         // position in the stack: push increments, replace keeps
}
```

New `Router` members:

```ts
readonly lane: () => string | null
readonly entryKey: () => string
/** Replace the current entry with `workspace` under `lane`. Not a history event.
 *  Reuses `key` if given so scroll memory finds the tab's place; returns the key. */
readonly switchLane: (lane: string | null, workspace: Workspace, key?: string) => string
/** Entries of this lane are dead from now on: popstate seeks past them. */
readonly forgetLane: (lane: string) => void
```

Rules the implementation follows:

1. Every `push` and `replace` in `commit` writes `{ key, lane: lane(), at }`.
2. The router keeps `Map<number, string | null>` from position to lane for entries this
   document created. Entries it did not create (from before a reload) have no row and
   count as dead when a lane is in force. History is therefore per document: a reload
   starts every tab's history empty. Say so in the docs.
3. On popstate with entry `e`: if `lane()` is null, or `e.lane === lane()`, behave as
   today. Otherwise decide direction as `sign(e.at - current.at)`. If a live entry of the
   current lane exists further in that direction, set a `seeking` flag and call
   `history.go(direction)`; while seeking, popstate does not touch the workspace or
   landings. If none exists, call `history.go(-direction)` with the flag set, so the
   person lands back where they were. When a same-lane entry is reached, clear the flag
   and apply it as a normal traversal, including scroll restore.
4. `forgetLane` rewrites that lane's rows to a reserved dead value.
5. `switchLane(null, …)` restores window semantics: every entry matches again.
6. Pure decision `seek(rows, currentAt, targetAt, lane)` returning `"apply" | "seek" |
   "bounce"` lives in its own module with unit tests. The handler is a thin caller.

Scroll memory is keyed by entry key and needs no change. The tabs plugin passes the tab's
stored key back to `switchLane` so a tab returns to its scroll position.

### The tabs plugin drives lanes like this

- On activation: `router.switchLane(front.id, front.workspace, front.key)` for the front
  tab from storage, or if there is no stored set, `switchLane(newId, router.workspace())`.
- `show(id)`: record `entryKey()` and the snapshot on the outgoing front tab, then
  `switchLane(id, incoming.workspace, incoming.key)` and store the returned key.
- `close(id)`: `forgetLane(id)`, then `show` the neighbour.
- On release, in this order: stop following the router and the preference, remove the
  strip and listeners, `switchLane(null, router.workspace())`, then the service is
  revoked and the state root disposed. Acquire in the reverse of that order so the Effect
  scope releases correctly.

### Persistence

One preference, `olai.tabs`, through `createPreference` from
`packages/web/src/client/preference.ts`, printed as JSON:

```json
{ "v": 1, "front": "t3", "tabs": [ { "id": "t1", "href": "/Tasks.olai", "title": "Tasks", "key": "…" } ] }
```

- Store hrefs, not workspaces. A tab brought to the front is re-parsed with
  `workspaceOf(router.routes, href)`, so a plugin that arrived or left since the tab was
  opened is honoured, and a href no plugin claims still opens as today's unknown address.
- The codec tolerates anything: no storage, an old version, a malformed record. Any
  failure yields the one-tab default. Test this.
- Write on every change of the list, the front, or the front tab's href. Read once at
  activation. Do not adopt writes from other browser windows live; last writer wins.
  Record this as a known limit in the docs.
- Entry keys stored across a reload are harmless: the router's memory is empty, so the
  restore is a no-op.

### The needs-you dot

An optional component of the tabs plugin, `attention`, needs `chat.state` and
`tabs.state`. It wears the dot on a tab when any leaf of its workspace points at a
conversation whose standing is `needs-you`, using the same predicate the Chats section
uses for "current" (route file equals the row's file and the row is unfolded, or the
route is that node). It reuses `LOOK["needs-you"].dot` for paint.

Chat publishes its roster as `chat.state` through `Offers.own("state", …)` but no
consumer names it yet, so there is no service tag for it. Add a static door,
`olai-plugin-chat/attention`, listed in chat's `olai.contracts`, exporting: a narrow
interface `Attention { rows: Accessor<ReadonlyArray<AttentionRow>>; at(node): … }`
declared in that file (not a type import of the browser `Roster`, so the fence's walk
finds no live module), `chatState = serviceTag<Attention>("chat.state")` in the way
`pins/src/contract.ts` declares `pinnedShelf`, plus `needing`, `LOOK`, and a pure
`isCurrent(route, row, unfolded)` extracted from `Chats`. The door must stay free of live
values; the fence test checks. No tab changes front because of this component. Without
the chat plugin the component waits and the strip has no dots.

### Pinned layouts open as tabs

In pins, add a component `tabs` with needs `[tabsState]` that holds the service in a
`heldService` holder for the plugin's faces. The pin's press handler asks the holder: if
a tabs service is held, `tabs.open(savedLayout(workspace))`; else the existing
`followLayout(router, workspace, event)`. Keep the browser-new-tab early return. Update
pins' docs.

### What is not per tab

Fold state of outline branches, the sidebar, the rail, the panel, the palette, and
appearance stay shared. The in-page filter and pane focus are already in the workspace
and so are per tab for free.

## Files and edits, by package

### `packages/plugins/navigation`

- `src/router.tsx`: `Entry` shape, position counter, lane map, `seeking` flag, the four
  new members, popstate branches. Keep the single `commit` funnel.
- `src/lanes.ts` (new): the pure `seek` decision and the lane row table type.
  `src/lanes.test.ts` alongside.
- `src/routes.ts`: add `label(route)` to `Routing`, implemented in `pages.ts` where the
  route faces are held.
- `src/palette/Palette.tsx` and `Shortcuts.tsx`: dispatch and list registered
  `app.keys` chords as described under Keyboard. Unit test the refusal.
- `package.json`: no change. `./workspace` (`hrefOfWorkspace`, `workspaceOf`,
  `savedLayout`, `lone`, `panesOf`), `./routes` (`HOME_ROUTE`) and `./layout-press` are
  already listed in `olai.contracts`; the tabs plugin imports through those.
- `docs.md`: a paragraph on lanes, what `switchLane` and `forgetLane` promise, that
  history is per document, that with no lane in force nothing changed, and that
  `app.keys` chords are now dispatched.
- e2e: extend `e2e/features/second_pane.feature` only if a pane verb's push/replace
  policy changed. It should not.

### `packages/plugins/layout`

- `src/index.ts`: `export const strip = location<() => JSX.Element>("layout.strip", "one")`.
- `src/Frame.tsx`: draw the seat inside `div.min-w-0.bg-paper` above `<Panes/>`, wrapped
  in an element with `data-testid="main-strip"`.
- `src/layout/css.ts`: publish `--height-strip` as `var(--height-tabs)` when the seat is
  filled and desktop, else `0px`; restore the prior inline value on cleanup as the other
  variables do. Add `--height-tabs: 2.625rem` to `packages/appearance/src/tokens.css`.
- `src/layout/sheet.ts`: subtract `var(--height-strip, 0px)` in `SHELL_SPLIT` and
  `SHELL_LONE`.
- `src/pane/label.ts`: delegate to `router.routes.label`.
- `src/browser.tsx`: add `strip` to the root contribution's `children`.
- `docs.md`: the seat, the variable, and that the seat is single occupancy.

### `packages/plugins/tabs` (new)

```
package.json          name olai-plugin-tabs; exports ".", "./browser", "./contract",
                      "./testids"; olai.contracts ["./contract", "./testids"];
                      dependencies naming everything src imports; devDependencies @olai/tests
tsconfig.json         the three-line one every plugin has
not-a-plugin.json     entries the fence test asks for (files that spell layout, pins, chat…)
docs.md               see Docs
src/index.ts          export const name = "tabs"; tabsState = serviceTag<TabsState>("tabs.state")
src/contract.ts       Tab, TabsState (verbs above), the stored shape and version
src/testids.ts        tabsStrip "tabs-strip", tabsTab "tabs-tab", tabsClose "tabs-close",
                      tabsNew "tabs-new", tabsDot "tabs-dot", tabsAddress "tabs-address",
                      tabsMenu "tabs-menu"; keys and values distinct from every other table
src/list.ts           pure list operations; list.test.ts
src/persist.ts        codec over parsedJson; persist.test.ts
src/store.ts          createTabs(router): the live state in one createRoot; follows
                      router.workspace(); drives switchLane/forgetLane; owns the preference
src/Strip.tsx         the strip, its menu, drag, address readout
src/links.ts          the document-level context menu on in-app links
src/browser.tsx       the row and its components
e2e/features/*.feature, e2e/steps/*.ts, e2e/selectors.ts
```

`src/browser.tsx` shape:

```ts
export default definePlugin({
  name,
  needs: [navigation, Offers, Slots],
  apply: Effect.gen(function* () {
    const router = yield* navigation
    const state = yield* Effect.acquireRelease(
      Effect.sync(() => createRoot((dispose) => ({ value: createTabs(router), dispose }))),
      ({ dispose }) => Effect.sync(dispose))
    yield* (yield* Offers).own("state", () => state.value)          // tabs.state
    yield* Effect.acquireRelease(Effect.sync(() => state.value.takeLane()),
      (release) => Effect.sync(release))                             // switchLane(id) … switchLane(null)
    yield* Effect.acquireRelease(Effect.sync(() => state.value.follow()),
      (stop) => Effect.sync(stop))                                   // router + preference
    yield* Effect.acquireRelease(Effect.sync(() => installLinkMenu(state.value, router)),
      (stop) => Effect.sync(stop))
    const slots = yield* Slots
    yield* slots.register("app.keys", …)   // one per chord; inert below the breakpoint
  }),
})
export const components = {
  strip: definePlugin({ name: "strip", needs: [tabsState, shell, rendererSlots], apply: … contribute(strip, () => <Strip/>) }),
  attention: definePlugin({ name: "attention", needs: [tabsState, chatState], apply: … }),
}
```

`layout.shell` is on the `strip` component, never on the row, as layout's own docs
require. The row activates with no renderer at all, like navigation does.

### `packages/plugins/pins`

- `src/browser.tsx`: component `tabs` holding `tabs.state`; `Pin.tsx` consults the holder
  before `followLayout`.
- `docs.md`: one sentence.

### `packages/plugins/chat`

- `src/attention.ts` (new static door): the `Attention` interface, `chatState` tag,
  `needing`, `LOOK`, and a new pure `isCurrent(route, row, unfolded)` extracted from
  `Chats`. The row's `apply` keeps publishing the same object; only the name gains a tag.
- `package.json`: door in `exports` and `olai.contracts`.
- `docs.md`: name the door.

### `packages/bundle`

- `olai.yml`: `- id: tabs`, `name: olai-plugin-tabs`, `section: This tab`, `quiet: true`,
  in the browser-only block after `layout`.
- `package.json`: `"olai-plugin-tabs": "workspace:*"`.
- Then `bun install`, `just regenerate-bun-nix`, and `bun test packages/bundle` until the
  fence, testids, composition and not-a-plugin claims are green.

### `packages/appearance`

- `tokens.css`: `--height-tabs: 2.625rem`.

## Docs, in the same PR

- `docs/index.md`: a Browser UI row for `plugins/tabs.md` with a gloss over forty
  characters.
- `docs/plugins/tabs.md`: symlink to `../../packages/plugins/tabs/docs.md`. The page
  says what a tab is, what is shared, every verb and chord, persistence and its
  last-writer limit, per-document history, phones, and what happens when tabs, chat, or
  pins are switched off.
- `docs/plugins/navigation.md`, `layout.md`, `pins.md`, `chat.md`: as listed above.
- `docs/editing.md`: a section "Keeping several pages open" for people.
- `docs/architecture/slot-ownership.md`: add `layout.strip` to the Layout row.
- `docs/architecture/e2e-coverage.md`: a Tabs domain row naming the feature files and
  scenario counts with an explicit `Open:` remainder, and a dated `## Tabs (#NNN)`
  section.
- `docs/architecture/e2e-economy.md`: rows for scenarios that overlap the unit tests
  (list operations, codec, seek), saying what the browser run additionally observes.
- `website/`: untouched. No picture there shows the main column.

## Tests

### Unit

- `navigation/src/lanes.test.ts`: apply, seek forward, seek back, bounce at either edge,
  dead lane skipped, unknown positions treated as dead, null lane applies everything.
- `tabs/src/list.test.ts`: open behind and in front, close front picks right then left,
  close last yields the front-page tab, closeOthers, duplicate placement, reorder bounds.
- `tabs/src/persist.test.ts`: round trip, missing storage, wrong version, garbage,
  duplicate ids, front not in the list.

### End to end (`packages/plugins/tabs/e2e/features`)

Selectors assert facts, never colour: `data-tab` (index), `data-tab-front`, `data-href`,
`data-tab-dot`. Every scenario ends with `And there should be no page errors`.

`strip.feature` (`@corpus:good`)
1. Opening the app shows one tab holding the address, in front.
2. Following a door changes the front tab's href and title; the tab count stays.
3. Open in new tab from a link's context menu adds a tab behind; the address bar does
   not change.
4. Clicking a tab brings it to the front; the address bar shows its href; the sidebar
   lights its door.
5. Closing the front tab shows its right neighbour; closing the rightmost shows the left.
6. Closing the last tab leaves a front-page tab.
7. Middle-click closes; the close button closes; Duplicate makes a copy right after;
   Close other tabs leaves one.
8. Drag reorders and the order survives a reload.
9. The next-tab chord shows the second tab; the new-tab chord opens; the close chord
   closes; all three are listed in the shortcuts sheet.
10. A split opened with Alt+click stays inside its tab; the tab title joins the leaves;
    closing a pane leaves the other page in the same tab.
11. The in-page filter typed in one tab is absent in another and back when returning.
12. Scroll position returns when a tab comes back to the front.

`history.feature` (`@corpus:good`)
1. Two moves in tab A, switch to B, one move in B: back lands on B's first page, not A's.
2. Back at B's edge bounces and stays on B; forward then works within B.
3. Close B; back from A skips B's entries.
4. Reload: the set returns, history is empty, back does nothing surprising.

`phone.feature` (`@corpus:good`, `@phone`)
1. No strip in the DOM; the pane strip still appears for a split.
2. Chords do nothing; Open in new tab navigates in place.
3. A set stored on desktop is still stored after a phone visit.

`attention.feature` (`@corpus:chat`)
1. A conversation that reaches needs-you puts a dot on its tab and only that tab.
2. Answering it clears the dot. The front tab does not change either time.

`pins.feature` (`@corpus:good`, `@share-scratch`)
1. Pressing a pinned layout opens it as a new tab.
2. With the tabs plugin switched off in the plugins panel, the same press opens in place.

`withdrawal.feature` (`@corpus:plugins` or the closest fixture with plugin switches)
1. Switching tabs off removes the strip, keeps the page, and back then walks the window
   history across what were tabs. Switching it back on restores the stored set.

Reuse step phrases from `packages/plugins/layout/e2e/steps/pane_steps.ts` for panes and
`world.open`, `showSidebar`, and the outline steps for doors. New steps live in
`packages/plugins/tabs/e2e/steps` and import the harness only through
`@olai/tests/harness/*`.

## Build order and commits

All of this work is one pull request against `master`, opened as a **draft** as soon as
the first step is committed and pushed, and kept a draft. The implementer never marks it
ready for review and never merges it: the maintainer does both. Each step below is one or
more commits on that PR's branch, and each leaves `just ci` green. Commit after each.

1. Navigation lanes, `Routing.label`, and `app.keys` dispatch, with unit tests and
   docs. No visible change until something registers a chord.
2. Layout seat, token, height variable, label delegate, docs.
3. Tabs package: contract, list, persist, store, the row without components, bundle
   wiring, docs symlink and index row. Typecheck and bundle tests green.
4. The strip component and `strip.feature`.
5. Lanes driven from the store and `history.feature`.
6. Keys and the link context menu; extend `strip.feature`.
7. Phone behaviour and `phone.feature`.
8. Chat door, attention component, `attention.feature`.
9. Pins component, `pins.feature`, `withdrawal.feature`.
10. `docs/editing.md`, `e2e-coverage.md`, `e2e-economy.md`, `slot-ownership.md`.

Use `just typecheck-fast-remote`, `just test-fast-remote`, `just e2e-fast-remote` while
iterating, and `just ci` on a clean pushed checkout before asking for review. Run
`just cordis-graph` after steps 3 and 9 and confirm the new edges are exactly the ones in
the ownership table above.

## Cordis checklist for the reviewer

- Every live value crosses a package as `tabs.state`, `navigation.state`, `layout.shell`,
  `chat.state`, or a contribution into `layout.strip`. No module-scope state in a shared
  door. The fence test enforces the doors; the graph shows the edges.
- The strip component names `layout.shell`; the row does not.
- Optional availability is by component: `tabs.attention` waits on chat, `pins.tabs`
  waits on tabs, and both hold the service in a holder that is cleared on release.
- Release order in the tabs row: listeners and strip, then `switchLane(null)`, then the
  service, then the root. The navigation plugin's own release is untouched.
- Lanes are inert when null, so the navigation plugin's behaviour without tabs is bit for
  bit what it was. `second_pane.feature` proves it.
- A tab records an href, never a route object, so a route survives its provider leaving
  and arriving, as the navigation docs promise for panes.
- No new timers. The only observer is the existing scroll memory. The context menu
  listener and the preference follow are acquired and released in the row's scope.

## Out of scope, recorded for later

- Live sync of the tab set between browser windows.
- A hover ⧉ on sidebar doors, which needs a change in each door's plugin.
- Dragging a tab out into a split and a pane up into a tab.
- Saving a tab set to the pinned shelf next to pinned layouts.
- A modifier-click that opens an olai tab.
- Palette rows that run a verb (new tab, close tab), which needs `AppPalette` to grow
  a `press` beside `href`.
