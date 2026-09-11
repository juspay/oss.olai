# Reminders: the trigger for due work, and the pull-only ruling

Vault copy of `REMINDERS-DESIGN.md` in olai worktree `.worktrees/reminders`, taken 2026-09-11 18:45 UTC after Fable's second amendment (one click listener dispatched by kind). The worktree file is the source; this copy is for the human to read from the vault.

Design for the roadmap node `reminders-notifications` ("Reminders: nothing ever alerts you about due work", under *Dates: journal, calendar & agenda* in `oss.olai`'s `projects/olai/roadmap.olai`). Written 2026-09-11 against `origin/master` at `5e087668e`, on branch `reminders`, and revised the same day on the human's two rulings (§13). Codex implements it as **one PR**; the human merges by hand. Nothing in this document has been implemented.

The 2026-08-24 review note on the node narrowed the scope: PR #358 shipped the alert **channel** (a notification seam over `@kolu/surface-app/notify`, a chime, an app badge, two preference rows, a served notification worker). What this design owns is the **trigger**, the **pull-only ruling**, and whatever the trigger needs from the channel.

## 1. Decisions at a glance

| Question | Decision |
| --- | --- |
| Pull-only? | **No.** Olai pushes exactly one thing about due work: a once-a-day reminder that what is owed is not empty, raised through the existing channel while the app is running. Everything else about due work stays pull. Written down in `docs/plugins/journal.md` (§10). |
| Which of the three triggers? | One rule that covers all three: **the first non-zero reading of what is owed, on each local day, in each browser.** "A date arriving" is the day rolling over with work on it; "a repeat coming due" is the next occurrence being a dated `todo` and therefore counted; "the agenda going non-empty" is the count going from zero to more than zero during the day. There is no separate repeat trigger, because the format gives a repeat no representation of its own (§3). |
| At what moment? | When the reader's `today` and the journal's `owed` reading first agree that `overdue + today > 0` for a day this browser has not yet been reminded about. That happens at local midnight for an open tab, on wake for a tab that slept through it, at boot for a tab opened later, or mid-day when the count leaves zero. |
| Quiet rule | **None about focus** (ruled by the human, 2026-09-11). A reminder fires whether or not the tab is in front of the reader: it is a once-a-day digest with content, not a banner about a form already on screen, so chat's "watched means nothing" doctrine does not carry over. The only quiet rules are the preferences and the per-day dedupe. Nothing in the reminders circuit reads focus or visibility. |
| Dedupe | One reminder per browser per local day, recorded under `olai.reminders.said` in `localStorage` **before** the alert is raised and followed across tabs on the `storage` event. The OS tag is `olai:due:<day>`, so two banners for one day replace rather than stack. A reload, a reconnect, a second tab opened later, and a second dated task the same day all raise nothing. |
| What it says | Title: what this deployment calls itself (`olai [box]`, the same word chat's banner wears). Body: `Agenda: 2 overdue, 3 on today`, the phrase the Agenda entry already speaks as its label, exported from `browser/agenda/owed.ts` and spelled once. The chime is best-effort: it plays only if the page has had a gesture, and a skipped chime is not replayed (§3.5). |
| Click | Opens `/agenda` in the focused pane. The payload is `{ kind: "due" }`, the second arm of the click union. It names no day, for the reason the union's header gives: what "it" is is a fact the app has when the press arrives. |
| Badge and tab mark | **Reminders never badge and never mark the tab.** The durable face of due work is the Agenda entry and rail dot, which already exist and already clear themselves. The badge stays chat's alone and keeps counting questions. |
| Knob | One browser preference, **Reminders** (on/off, default on), drawn as a third row under Alerts and Alert sound and frozen when Alerts is off, exactly as Alert sound is. It is a browser preference and not a `Config` knob because it answers "how does this browser alert me", the door the other two rows already use (§6). |
| Ownership | The channel leaves the chat row for a new tab-only row, **`alerts`**, which offers `alerts.channel`; chat's attention circuit and the journal's reminders both name it. Ratified by the human, 2026-09-11 (§7, §13). |

## 2. What is actually there today

Facts the design rests on, so a reviewer can check them rather than trust them.

- **The channel is not app-root any more.** #358 put `notify.ts` at the client's root. #557 (the Cordis audit, phase 2) moved it to `packages/plugins/chat/src/browser/notify.ts` and wrote the reason in its header: a live value on a general package's door with one row behind it fails the audit's §12, and "the honest fix for a helper nobody else uses is to put it behind the wall of the row that owns it". The header also says the union has "ONE ARM today ... because the second arm is what the shape is for". This design is that second row, so the seam has to come back out to an owner both rows can name. `docs/architecture/overview.md` still says `web/src/client/notify.ts`; that sentence is stale and is one of the docs to fix.
- **What the channel is, file by file, all under `packages/plugins/chat/src/browser/`:** `notify.ts` (the seam, the `NotifyClick` union, the permission mirror, `followNotifications` with a module-private holder), `alerts.ts` (the Alerts and Alert sound preferences, `tabWaiting`, the `chat.alerts` service tag and its holder), `alerts/keys.ts` (the two `localStorage` keys, exported as `olai-plugin-chat/alert-keys` and imported by `preferences_steps.ts`, `suite.testlib.ts` and the fence), `AlertRows.tsx` (the two rows and the Allow notifications button), `chat/attention/chime.ts` (two oscillators, first-gesture unlock, one audio context), `chat/attention/badge.ts` (`setAppBadge` on an installed icon, else the tab mark through `theme.appearance`'s `chrome.waiting`). The `tab-attention` component in `browser.tsx` is the one writer of `chrome.waiting`.
- **What is chat's own and stays chat's:** `attention/attention.ts` (the circuit), `alarm.ts` (the two-line rule), `watching.ts` and `elsewhere.ts` (visible, focused, panel open, and the cross-tab beat on `olai.chat.watched`), `notice.ts` (the banner's words), `reveal.ts` and `asked.ts` (what a press opens).
- **The reading a reminder is about already exists.** `packages/format/src/agenda.ts`'s `owedNow(derived, today)` answers `{ overdue, today }` off the `owedByDay` index. The journal's server streams it as `owed` (`packages/plugins/journal/src/server.ts`), re-read per published revision and sent only when it changed by value. The browser subscribes through `createOwed(today)` in `packages/plugins/journal/src/browser/dates.ts`, which re-opens the subscription when `today` moves and answers `undefined` before the first frame. The Agenda entry and rail dot draw from that same reading (`browser/agenda/owed.ts`). The count is the one the Agenda page is drawn from, held to it by `owed.index.test.ts`.
- **The day is the renderer's clock.** `ui-renderer.clocks` offers `today()`, an accessor that moves at the next computed local midnight and whenever the page becomes visible again (`packages/web/src/client/clock.ts`). The journal holds it per activation in `browser/clock.ts`. Nothing in this design needs a timer of its own.
- **A repeat has no representation of its own.** Completing a repeating node captures the next occurrence as a fresh dated `todo` node. "Where it is drawn: nowhere special" (`docs/format.md`, *Repeating*). Nothing on disk says "a repeat is coming"; the only fact is a dated `todo`, which `owedByDay` already counts.
- **A tab-only row is an ordinary row.** `theme`, `preferences` and `layout` are declared in `packages/bundle/olai.yml` with `name: olai-plugin-<x>` (the package root), `section: This tab`, `quiet: true`, and offer browser services with `Offers.own` (theme offers `theme.appearance` from `browser.tsx`).
- **Browser preferences** are `createPreference` in `packages/web/src/client/preference.ts`: read into a signal, written back, followed on the `storage` event, tolerant of storage that throws. Keys are `olai.`-prefixed.
- **Navigation from outside a component** is the `navigation.state` service (`olai-plugin-navigation`'s `navigation` tag), which extends `Router` and carries `go(route)`; `agendaRoute` is `packages/plugins/journal/src/browser/routes.ts`'s `agenda.to({})`.
- **The deployment's word** for a banner title arrives on `layout.deployment` (`olai-plugin-layout/contract`'s `deployment`), held per activation; chat's `deployment` component and `browser/deployment.ts` show the shape.
- **Preference rows** are contributed to `preferences.sections` (`olai-plugin-preferences/contract`'s `sections`) through `rendererSlots.contribute`, drawn in bundle order.
- **The e2e stage for alerts** is `packages/tests/support/alerts.ts` (wraps `showNotification` and the oscillator's `start`, makes `Notification.permission` say what the context was set up with) under the `@alerts` and `@alerts-denied` tags in `support/hooks.ts`, and `step_definitions/alerts_steps.ts` delivers a press as the worker's own message. `the_agent_waits_on_you.feature` says plainly that headless Chromium reports every page focused and visible, so "a backgrounded window" was not a state that harness could produce. This design never needs that state (§3.2, §9.2).

## 3. The trigger rule

### 3.1 Why the three candidates are one rule

The roadmap offered three triggers. Read against the format, they are one reading looked at three ways:

- **A date arriving.** A node's `date` becomes today, or yesterday's unfinished work becomes overdue, when the local day rolls over. Both move `owedNow(derived, today)`: the `today` bucket and the `overdue` sum.
- **A repeat coming due.** There is no such event. The next occurrence is a fresh node with a `date` and `todo`, minted at the moment the previous one was completed, dated one period after the previous node's own date. It is counted by `owedByDay` from the moment it exists, and it becomes today's or overdue work exactly as any other dated task does.
- **The agenda going non-empty.** Taken literally this is useless: a vault with one chronic overdue item never goes non-empty again, so a task dated today would never alert. Read as "the day's owed count leaving zero", it is the mid-day half of the first candidate.

So the reminder is about **the day's reading**: `owedNow` for the reader's own `today`, the same two integers the Agenda entry burns from.

### 3.2 The rule, whole

Written as arithmetic first, so it lives in a pure module with no browser under it (the shape of `alarm.ts`).

```
inputs:  day      the reader's today (ISO day), or "" with no clock
         owed     { overdue, today } or undefined before the first frame of this subscription
         said     the day this browser last reminded about, or null
         on       the Reminders preference AND the Alerts preference

due     = owed !== undefined && owed.overdue + owed.today > 0
fresh   = day !== "" && said !== day
consider = due && fresh

if !consider        -> nothing, say nothing
if consider && !on  -> nothing, say nothing            (a switched-off day is not spent)
if consider && on   -> say day, then raise: chime, banner
```

Three consequences worth stating:

- **Once per day per browser.** After `said === day`, nothing this day can raise a second reminder: a second task dated today, a reconnect that re-snapshots the stream, a reload, a second tab. A person who hears "3 on today" at nine and adds a fourth at two is not told about the thing they just typed.
- **A day switched off is not spent.** With Reminders (or Alerts) off, `said` is not written. Turning it back on the same day reminds once. This is the honest reading of a switch: off means "do not tell me", not "pretend you told me".
- **Focus is not a fact this rule reads** (ruled by the human, 2026-09-11, reversing the first draft). Chat's "the form appearing is the alert" is about a form that is already on screen; a reminder is a daily digest that says something the screen does not, and a reader with olai in front of them is told once like anybody else. So the reminders circuit never reads `document.hasFocus()`, `visibilityState`, or chat's cross-tab beat, and the only things that keep it quiet are the two preferences and `said`.

### 3.3 When the rule runs

The reminders circuit is one `createEffect` over three signals: `today()` from the clock, `owed()` from `createOwed(today)`, and `said()` from the preference. It runs whenever any of them moves. That is enough to cover every moment the roadmap named, with no timer of its own:

| Moment | What moves | What happens |
| --- | --- | --- |
| Local midnight, tab open on screen | `today()` (the clock's timer); `createOwed` re-opens the stream; the new frame arrives | first non-zero frame of the new day raises |
| Laptop opened at nine | `today()` re-read on `visibilitychange`; stream re-opens | same |
| Installed app launched at login, window behind | boot; first frame | first non-zero frame raises |
| A task dated today is written mid-day, count 0 → 1 | `owed()` | first non-zero frame of the day |
| A repeat completed, next occurrence dated today or earlier | `owed()` | same |
| Reconnect (roughly one second `pending`) | `owed()` goes `undefined` then returns | `said === day`, nothing |
| Reload, or a second tab | fresh circuit reads `said === day` from storage | nothing |
| Journal switched off then on | the component leaves and returns; `said` unchanged | nothing the same day |

The first reading of a fresh circuit **is** allowed to raise, unlike chat's `alarmFor` where `was === undefined` never alerts. Chat's rule guards against ringing for questions asked while the tab did not exist; here the guard is `said`, which is per browser and survives the tab. A background launch at login is exactly the case that should ring.

### 3.4 Cross-tab races, stated

Two open tabs of one olai can both receive the same frame at the same instant, both read `said !== day`, and both raise. The banner is one (same tag, the OS replaces); the chime may sound twice. This is the residual `elsewhere.ts` accepted for chat in its own words ("electing one to speak for the rest is a different mechanism"). The `storage` event closes the window for every later moment. The design does not add an election, and the reviewer should not ask for one.

What the design does promise, and the tests pin: a tab that opens **after** the reminder, and a tab that **reloads** after it, raise nothing.

### 3.5 The chime is best-effort, and the notification is the reminder

Ruled 2026-09-11 on Codex's finding. The browser's autoplay rule means an `AudioContext` opens only inside a pointer or keyboard gesture; `chime.ts` takes the first gesture the page gets and, before one, `chime()` returns without playing and grumbles once. A reminder raised at boot, or on a frame that arrives before anybody has touched the page, therefore has no sound available to it.

The rule:

- **The notification is the reminder.** It is raised whenever §3.2 says raise, gesture or no gesture, and `said` is written before it as before.
- **The chime is the second half of it, offered when it can be.** It plays if the audio context is already unlocked, and is skipped otherwise. A skipped chime is not owed later: nothing remembers it, nothing replays it at the next gesture, and the day stays said.

Why not replay at the first gesture: the first gesture of the day is almost always a click into the app, which is a reader already looking at it, and a chime at that moment is a sound about a banner they have had for an hour. It would also give the reminder a second piece of per-day state to keep in step with `said`. Why not change the audio policy: it is the platform's, and the channel already answers it the honest way (`grumble`, no throw).

What this costs is stated in the docs: on a tab nobody has clicked since it opened, the daily reminder is a notification without a sound. The chat alerts have the same limit and say so in their hint ("The first plays only after you click the page").

## 4. What it says, and what a press opens

**The notice.**

```ts
{
  tag:   `olai:due:${day}`,
  title: called ?? "olai",                       // layout.deployment's word, as chat's banner
  body:  `Agenda: ${phraseOf(owed)}`,            // "Agenda: 2 overdue, 3 on today"
  data:  { kind: "due" },
}
```

`phraseOf(owed)` is `browser/agenda/owed.ts`'s private `said` with its "Agenda — " prefix taken off and the function exported: the words "2 overdue, 3 on today" are then spelled once for the entry's label, its title, the rail's label and the banner. The entry keeps its own prefix. No titles of nodes ride the banner: the `owed` stream is two integers, and the reading that has titles is the whole agenda page, which a reminder has no business subscribing to.

**The chime** is the channel's, gated by Alert sound, and is asked for at the same moment the banner is; it plays only if the page has already had a gesture (§3.5). Both are raised untracked, after `said` is written, notification first.

**The press.** The worker focuses or opens the window (the framework's half). The page's half is `navigation.state`'s `go(agendaRoute)`: the focused pane shows `/agenda`. Nothing scrolls and nothing waits for rows: the agenda's own page is the answer. A cold-start press (no window open) is handed over by the seam at startup and takes the same path. A press on a reminder from a previous day opens today's agenda, which is the right answer to "take me to it".

**The click union.** `NotifyClick` becomes `{ kind: "ask" } | { kind: "due" }`. The validator accepts both arms and refuses everything else. A stale pre-upgrade envelope is dropped as before.

**One listener, dispatched by kind** (ruled 2026-09-11 on Codex's second finding, §7). The framework's `onClick` claims each click durably before it calls its handler and consumes the cold-start URL payload at the first subscription, so two subscribers on the seam would race: whichever ran first would claim a click of the other's kind and the other would never see it. So the alerts row holds the **single** `seam.onClick` for its activation and dispatches by `data.kind` to per-kind subscribers: chat subscribes for `ask`, the journal for `due`, each through `channel.onPress(kind, handler)`. A click whose kind has no subscriber yet is **held** by the row (one per kind, the newest replacing an older one, since a press means "take me to it" and two presses of one kind mean the same thing) and delivered the moment that kind subscribes, within the channel activation's lifetime. That is what makes a cold-start `due` press work: the seam hands the payload over at startup, before either row's browser half has mounted, and the journal's component takes it when it comes up. A held click leaves with the channel activation and is not persisted anywhere.

## 5. The devices reminders do not use

**No badge, no tab mark.** The badge exists for chat because a shut panel has no face on screen, and it counts questions; a `4` on the dock that meant "4 overdue" one hour and "4 questions" the next would be a number nobody could read. Due work already has a durable face that clears itself, the Agenda entry, so the reminder is an event only. This also keeps `chrome.waiting` a single-writer fact and keeps `wear` a single-claimant device, which is what lets the channel move without growing a claim table (§7).

## 6. The knob and its door

**One row: Reminders.** On/off, default **on**, key `olai.reminders`, a `createPreference` with `boolCodec(true)`. Drawn by the journal as a contribution to `preferences.sections`, under the two alert rows, using `@olai/ui-primitives`' `Row` and `Segmented` exactly as `AlertRows.tsx` does. Frozen (drawn inert, not hidden) while Alerts is off, like Alert sound, with the hint saying why.

Hints, read off the choice in force:

- on, alerts on: "Once a day, a notification says what is overdue and on today, with a chime if you have clicked the page since it opened."
- off: "Nothing says the day has work on it. The Agenda entry still shows it."
- alerts off: "Alerts are off, so nothing will remind you."

**Why a browser preference and not a `Config` knob.** `docs/running.md` names three settings doors: the vault file for policy and behaviour shared by every consumer of the directory, the environment for machine resources, and the browser for "how this tab reads". Alerts and Alert sound went through the third door because whether *this device* rings is a fact about the device: a phone and a laptop on the same vault are entitled to differ. Reminders is the same question one row down. A `Config` knob on the `journal` node would also have to travel to the browser on a new wire member, which nothing else in the journal needs and which would put a per-device choice in a file shared by every device. The default is on to match #358's ruled default for alerts; a reader who never wants it has one press, and Alerts off already silences it.

The `said` record (`olai.reminders.said`, an ISO day or absent) is the journal's own, under the same door, and is not a row: it is a fact the browser keeps, not a choice a reader makes.

## 7. Riding the channel: who owns it now

The seam sits behind the chat row's wall with a comment saying it moves the day a second row opens it. The journal is the second row. Two ways to open it:

**(A) Recommended: a tab-only row `alerts` owns the channel.** New package `packages/plugins/alerts` (`olai-plugin-alerts`), declared in `olai.yml` as `name: olai-plugin-alerts`, `section: This tab`, `quiet: true`, placed before `chat` and `journal` in the roster. It owns, moved from chat with their headers intact: the seam (`notify.ts`), the two preferences and `tabWaiting` (`alerts.ts`), the keys (`keys.ts`, re-exported as `olai-plugin-alerts/keys`), the two rows and the Allow button (`AlertRows.tsx`), the chime (`chime.ts`), the badge (`badge.ts`), and the `tab-attention` component that writes `chrome.waiting`. It offers **`alerts.channel`**:

```ts
export interface Channel {
  // the two choices, and the tab's half of the badge
  readonly alertsOn: Accessor<boolean>;      readonly setAlertsOn: (v: boolean) => void
  readonly alertSoundOn: Accessor<boolean>;  readonly setAlertSoundOn: (v: boolean) => void
  // the seam
  readonly consent: Accessor<Consent>
  readonly ask: (force?: boolean) => Promise<void>
  readonly notify: (notice: Notice) => Promise<void>       // no-op while Alerts is off
  // ONE subscriber per kind, claimed atomically: a second claimant for a kind is refused
  // with a sentence naming both, and the claim is released by the returned function or the
  // subscriber's scope. A click of a kind nobody has claimed is held (newest per kind) and
  // delivered on claim; the row keeps the one framework listener (§4, §8).
  readonly onPress: <K extends NotifyClick["kind"]>(
    kind: K, press: (asked: Extract<NotifyClick, { kind: K }>) => void,
  ) => () => void
  // the devices
  readonly chime: () => void                               // no-op while Alerts or Alert sound is off
  readonly wear: (count: number) => void                   // app badge or tab mark; clamped to 0 while Alerts is off, and put back to 0 when Alerts is switched off
}
export const alertsChannel = serviceTag<Channel>("alerts.channel")
```

The row also owns the **one** framework click listener and the per-kind dispatch with its held clicks (§4, §8): `followNotifications` subscribes `seam.onClick` once, before the offer, and `onPress(kind, …)` is how a consumer takes its arm. The "Alerts off is off for all three devices, and the icon is put back" rule moves into the channel and is spelled once; chat's circuit stops checking the preferences itself. `NotifyClick`, `Notice`, `Consent` and the `alertsChannel` tag are the row's contract door (`olai-plugin-alerts/contract`, a static import both rows may take). The row has no server half and no wire member.

Chat then names `alerts.channel` on a **component** (`attention`), not on the row, so the conversation keeps running with the alerts row switched off. That component holds the channel by identity for its activation and mounts the circuit itself (`createRoot` in its `apply`, the shape `tab-attention` already uses), reading the conversation through a row-private holder; `Panel.tsx` no longer calls `createAttention`. Chat's `alerts`, `alert-controls` and `tab-attention` components go. `chat.alerts` is no longer offered.

**(B) Fallback: the journal names `chat.alerts`.** Widen chat's offered `Alerts` shape with `notify`, `onPress`, `chime`, and have the journal's reminders component name `chat.alerts`. Smaller diff, wrong owner: a serve with chat off (no agent, `on: no`) would show the journal waiting on a chat key and would never remind, and the badge and permission seam would be a chat fact that reminders borrow. The audit's §12 argument in `notify.ts`'s own header rules this out for a second row.

The design is written for (A). If the human picks (B), §11's step 1 is replaced by widening `chat.alerts` and the rest stands.

## 8. Cordis ownership

| Owner | What it owns | Acquired | Released when the owner leaves | Absence, seen from the other side |
| --- | --- | --- | --- | --- |
| `alerts` row, `channel` component | the **one** framework click listener (`seam.onClick`, subscribed once in `followNotifications`) and the permission listener, the per-kind subscriber table and the held clicks (at most one per kind), the two preference followers (`storage` listeners), the audio context and its first-gesture listeners, the badge's `worn` count | `Effect.acquireRelease` in `apply`, offered with `Offers.own("channel")` after acquisition; the framework listener is subscribed before the offer, so a cold-start payload is consumed into the held table before any consumer can ask | listener removed, subscriber table cleared, held clicks dropped, setters invalidated, audio context closed, badge cleared (`wear(0)`), offer withdrawn | consumers of `alerts.channel` read `waiting`; chat's questions still mark the header toggle; nothing rings; a press with the row off reaches nobody (the framework's own fallback navigation still focuses the window) |
| `alerts` row, `tab-attention` component | the one write to `theme.appearance`'s `chrome.waiting` | needs `theme.appearance` and `alerts.channel` | `chrome.waiting(false)` then dispose | tab wears no mark |
| `alerts` row, `controls` component | the two rows in `preferences.sections` | `rendererSlots.contribute` | withdrawn | the preferences panel has no alert rows |
| `chat` row, `attention` component | the attention circuit: the fold over the cell, `createWatching`'s listeners and beat, `createElsewhere`'s channel, the `ask` claim on `channel.onPress` | needs `alerts.channel`, `theme.appearance`-free; holds the channel by identity | circuit disposed, `wear(0)`, the `ask` claim released, `BroadcastChannel` closed | with the alerts row off, the component waits; the panel, toggle and forms are untouched |
| `journal` row, `reminders` component | the reminders circuit (one effect), the `olai.reminders` and `olai.reminders.said` preference followers, the `due` claim on `channel.onPress` | needs `alerts.channel`, `navigation.state`, `layout.deployment`; shares the row's held clock and wire (`holdClocks`, `holdJournalWire`) | effect disposed, followers stopped, the `due` claim released; `said` stays in storage; a `due` click arriving afterwards is held by the alerts row until the component returns |  with the alerts row off: waits, no reminders, calendar and agenda unaffected; with the journal off: no reading, so nothing; with no clock: `today()` is `""`, nothing is asked |
| `journal` row, `reminder-controls` component | the Reminders row in `preferences.sections` | needs `alerts.channel` (to freeze on Alerts off) and `rendererSlots` | withdrawn | no Reminders row while the alerts row is off |

Rules the reviewer holds each of these to:

- **No timer of its own.** The day comes from `ui-renderer.clocks`; the reading is a wire subscription the server re-reads per revision. A `setTimeout` or `setInterval` under `journal/src/browser/reminders/` is a defect (§9 pins it with a sweep).
- **The subscription is the row's.** `createOwed` reads `journalWire()`, the row's held client; a component shares the row's holders and adds none for the wire.
- **The claim before the alert.** `said` is written before `notify` and `chime` are called, so an exception on the way out cannot leave a day both unsaid and half-raised.
- **No reading of focus.** The reminders circuit subscribes to nothing about the window: no `focus`, `blur` or `visibilitychange` listener, no `hasFocus()`, no `BroadcastChannel`. What it needs about the tab it gets from the clock and the wire.
- **Holders are the consumer's, cleared by identity.** Chat and the journal each keep their own `heldService` of `alerts.channel`; neither reads the provider's module state.
- **The seam is one per origin, and the listener is one per activation.** `followNotifications` throws on a second activation, as today, and only the alerts row calls it; it is the only caller of the framework's `onClick` in the tree, and it calls it once. Neither chat nor the journal imports `@kolu/surface-app/notify`.
- **A kind has one owner.** `onPress(kind, …)` refuses a second claimant for the same kind in one synchronous step, naming both; the refused claim installs nothing and its release removes nothing of the winner's. A held click is delivered to the claimant exactly once, on the claim, and is then forgotten.
- **Declared on components, never on rows.** Neither chat's nor the journal's row-level `needs` names `alerts.channel`.

## 9. What the reviewer checks, and the tests that pin it

### 9.1 Unit (`bun test`)

- `journal/src/browser/reminders/rule.test.ts`: the table in §3.2, every row, including `day === ""`, `owed === undefined`, both counts zero, `said === day`, off-and-not-spent, and the positive row for overdue only, today only, both.
- `journal/src/browser/reminders/notice.test.ts`: tag per day, title falls back to `olai`, body phrase for overdue only, today only, both.
- `journal/src/browser/agenda/owed.test.ts`: `phraseOf` and the entry's `said` agree word for word (one spelling).
- `alerts/src/notify.test.ts` (moved from chat): the validator accepts `ask` and `due`, refuses `{}`, a string, and an unknown kind.
- `alerts/src/presses.test.ts` (new, over a fake seam whose `onClick` is counted): the seam is subscribed exactly once however many kinds claim; a `due` click with `ask` claimed first reaches the `due` claimant and not the `ask` one, and the other way round; a click arriving with no claimant for its kind is delivered once when that kind is claimed and not again; two such clicks before a claim deliver only the newer; a second claimant for one kind is refused and the first still receives; releasing a claim and claiming again delivers a click that arrived in between; disposing the channel drops held clicks and unsubscribes the seam.
- `alerts/src/channel.test.ts`: `notify` and `chime` are no-ops while Alerts is off; `chime` while Alert sound is off; `wear` clamps to 0 while Alerts is off and puts 0 back when Alerts flips off with a count worn; `chime()` before any gesture returns without throwing and starts no oscillator (`chime.test.ts` already covers the unlock; this pins the call before it).
- `journal/src/browser/reminders/circuit.test.ts`: over fake signals and a fake channel, the order of calls is `said` write, `notify`, `chime`; a `chime` that does nothing leaves `said` written and is not called again on the next frame of the same day.
- `journal/src/browser/claims.test.ts` (new, the shape of `chat/src/browser/claims.test.ts`): no file under `browser/reminders/` spells `setTimeout(`, `setInterval(`, `hasFocus`, `visibilityState` or `BroadcastChannel`; the only speller of `olai:due:` is `reminders/notice.ts`; the only writer of `olai.reminders.said` is `reminders/said.ts`.
- Existing: `chat/attention/alarm.test.ts`, `notice.test.ts`, `reveal.test.ts`, `badge.test.ts`, `chime.test.ts` unchanged in substance; `badge.test.ts` and `chime.test.ts` move with their modules. `elsewhere.browsertest.ts` and `asked.browsertest.ts` stay in chat.
- `@olai/bundle`: `fence.test.ts` gains the alerts package in its tenant and door lists; `plugin_docs.test.ts` requires `docs/plugins/alerts.md`; `testids` collision check covers the moved `prefsAllowNotify`.

### 9.2 End to end (Cucumber + Playwright)

**No new stage device.** The first draft needed a window that is behind, which headless Chromium cannot produce, and proposed a `@behind` tag and two window steps faking `document.hasFocus` at the last inch. With the focus rule gone (§3.2), nothing in the reminders circuit reads focus, so nothing needs to fake it: every scenario runs in the ordinary focused page, and the existing `@alerts` stage (`support/alerts.ts`, `hooks.ts`, `alerts_steps.ts`) is enough as it stands. Chat's feature keeps its shut-panel way of being unwatched. No `hasFocus` wrapper, no tag, no window steps; a reviewer who finds one in the PR should ask what it is for.

New feature file `packages/tests/features/the_day_reminds_you.feature`, every scenario `@scratch:...` and `@alerts` (or `@alerts-denied`), each ending `And there should be no page errors`:

1. **With work owed, the day is announced once, at boot, without a sound** (a corpus with overdue work such as `good`, whose `order` node is dated 2026-08-10): a notification whose body says `overdue`, tagged `olai:due:<today>` (the step computes today as `isoDayOf(new Date())`, the way `a task is due today` already does), **no chime rang** (the page has had no gesture, §3.5), the tab says nothing is waiting, and the Agenda entry burns; then `I click the page` (a gesture) and still no chime rang, because a skipped chime is not replayed.
2. **Work that becomes due during the day, after a gesture, chimes** (a scratch with nothing owed, `I click the page` first, then `a task is due today`): a notification says `1 on today`, the chime rang, the Agenda entry burns. The gesture step is the one the chat feature relies on implicitly (its scenarios press Send); here it is written out, because the chime is what the scenario is about. `I click the page` is a new step in `reminders_steps.ts` (a real pointer click on the page body; the existing `I click the page {string}` in `html_steps.ts` names a file and is not it).
3. **Once a day** (after 2): a second task dated today raises no second notification and no second chime (`support/alerts.ts` counts both), and the entry's count moves to 2.
4. **A reload says nothing again** (after 2): reload, the worker ready, no notification, no chime.
5. **A second tab says nothing** (after 2): open a second page in the context, the worker ready, no notification in it.
6. **Pressing it opens the agenda, with chat listening too** (after 2, with the chat row on and its panel mounted, which is the ordinary stage): `the notification is pressed` (the existing step, sending the `due` payload with a fresh id and the ackable source), the pane shows the agenda page and the agent panel does not open. This is the two-consumer pin: chat has claimed `ask` on the same channel and the click still reaches the journal.
6b. **A press that opened the window still opens the agenda** (cold start): a step `I open the app from a pressed reminder` boots the page at the app URL carrying the framework's cold-start params (the data param holding `{"kind":"due"}` and a fresh click id, spelled in the step the way `PRESS_TYPE` is, for the same reason); once the journal's browser half is up, the pane shows the agenda page, the params are gone from the address bar, and a reload of that address does not open the agenda a second time (the framework strips them and dedups the id). This pins the hold: the seam consumed the payload at boot, before the journal's component existed.
7. **Reminders off is off, and off does not spend the day** (`I set Reminders to "off"`, a task due today): no notification, no chime, the Agenda entry still burns, `this browser has stored that reminders are "off"`; then `I set Reminders to "on"`: the reminder arrives.
8. **Alerts off freezes the Reminders row**: `I set Alerts to "off"`, the Reminders row cannot be set and explains itself; and a task due today raises nothing.
9. **A browser that refused notifications still chimes** (`@alerts-denied`, a scratch with nothing owed, `I click the page` first, then `a task is due today`): no notification, the chime rang, no tab mark. Driven after a gesture rather than at boot for §3.5's reason; the boot half of the denied case is that no notification is raised and nothing throws, which scenario 1's corpus under `@alerts-denied` may assert as a second example if Codex finds it cheap.
10. **The journal off takes the row and the reminder with it** (switch `journal` off on the plugins panel, a task due today: nothing; switch it on: the reminder arrives, the Reminders row is back).
11. **The alerts row off leaves the calendar standing** (`@rows-off:alerts`): the plugins panel says `journal` waits on `alerts.channel` for its reminders component, the preferences panel has no Alerts rows, the calendar and the agenda page still draw, a dated task still lights its day.

Existing files that must pass unchanged in substance: `the_agent_waits_on_you.feature` (all seven), `preferences.feature`, `preference_storage.feature`, `agenda.feature`, `journal_plugin.feature`, `chat_plugin_lifecycle.feature`, `the_plugin_switch.feature`. Step names for the Alerts rows keep working because the rows keep their `pref` attributes and testids.

### 9.3 Reviewer's checklist

- The pure rule file has no import from `solid-js` or the DOM.
- `said` is written before `notify` and `chime` are called, in that order, in one place.
- Nothing in `reminders/` calls `wear`, `setTabWaiting` or `chrome.waiting`.
- Nothing in `reminders/` reads focus or visibility, and the test package gained no `hasFocus` wrapper, no `@behind` tag and no window steps.
- The chime is called after `notify`, never gates `said`, and nothing in `reminders/` or the channel remembers a chime that did not play. `chime.ts`'s unlock is unchanged.
- Every e2e scenario that asserts `the chime rang` supplies a gesture before the frame that raises; every scenario that raises at boot asserts `no chime rang`.
- The banner names no node and no day in its `data`.
- `@kolu/surface-app/notify` is imported by the alerts row only, `onClick` is called once per activation, and both chat's and the journal's press handling go through `channel.onPress(kind, …)`. Chat's press behaviour (open the panel, reveal the waiting form) is unchanged, and `the_agent_waits_on_you.feature`'s press scenario passes without edits.
- The Reminders row is frozen, not hidden, with Alerts off.
- Chat's e2e press step still walks the full handshake (id, ackable source, durable claim) and the reminders press step reuses it with the other payload rather than a second implementation.
- The `alerts` row is declared before `chat` and `journal` in `olai.yml`, and both name `alerts.channel` on a component.
- `docs/architecture/overview.md` no longer names `web/src/client/notify.ts`.

## 10. The pull-only ruling, as it will be written

To go into `docs/plugins/journal.md` under a new heading **Reminders**, and in one sentence in `docs/architecture/overview.md`'s journal section:

> **Olai is pull with one push.** Dates, repeats and the agenda are readings: nothing on disk is a reminder, nothing schedules one, and nothing fires at a time of day. The one thing olai says unprompted about due work is a reminder that the day has work on it: once per day, in each browser, the first time that day's reading of what is owed is not empty, a chime and a notification say how much is overdue and how much is on today, whether or not olai is in front of you, and pressing it opens the agenda. It rides the same channel and the same limit as the agent's alerts: the app has to be running, foreground or background, and a closed olai hears nothing. There are no per-node alarms, no times on dates, no snooze, no repeat-specific alert, and no push server; each of those is its own decision rather than a detail of this one.

The roadmap node in `oss.olai` is the human's to update; this document is the text to update it from.

## 11. Docs to update, in the same PR

| File | Change |
| --- | --- |
| `docs/plugins/alerts.md` (new; symlink of the package's `docs.md`) | the row: what it owns, `alerts.channel`, the two rows, the click union and its arms, the honest limit; the ownership move from chat |
| `docs/index.md` | a row for `plugins/alerts.md` under Browser UI |
| `docs/plugins/journal.md` | **Reminders** section: the rule (§3.2 as prose), the moments, the once-a-day dedupe and that focus is not read, the knob, the pull-only ruling (§10), the `reminders` component and what leaves with it |
| `docs/plugins/chat.md` | the seat table's `preferences.sections` row moves to the alerts row; the ownership paragraph about permission state and click listeners moves with it; chat names `alerts.channel` on its `attention` component |
| `docs/chat.md` | the alerts section: the rows now belong to the alerts row and there are three; a reminder is not a question and does not badge |
| `docs/architecture/overview.md` | the browser paragraph on the seam (`web/src/client/notify.ts` is stale); the journal paragraph gains one sentence on reminders and the ruling |
| `docs/architecture/e2e-coverage.md` | the *Dates and journal* row gains `the_day_reminds_you`; the *Preferences and identity* row names the Reminders row |
| `packages/tests/README.md` | not touched: the alerts stage fakes nothing new |
| `website/` | not touched: the reason to try olai has not changed and no picture is now wrong |

## 12. The plan for Codex: one PR, one green commit per step

**One PR, and only one.** The five steps below are five commits on the `reminders` branch, never five pull requests, and there is no follow-up pull request: the channel move, the trigger, the feature file and the docs all open together. Each step ends with `just typecheck-fast-remote` and `just test-fast-remote` green; steps 4 and 5 also `just e2e-fast-remote`. Push and open **one PR** at the end; `just ci` on the final head. Never rebase; merge master in if it moves.

1. **Move the channel out of chat into a new `alerts` row.** Create `packages/plugins/alerts` (package.json with `.`, `./contract`, `./browser`, `./keys`, `./testids`, `./all.css` if needed; `docs.md`; `src/testids.ts`). Move `notify.ts`, `alerts.ts`, `alerts/keys.ts`, `AlertRows.tsx`, `chat/attention/chime.ts`, `chat/attention/badge.ts` and their tests, headers intact, adding only the paragraph that says why they moved. Define `Channel` and `alertsChannel` in the contract door; fold the Alerts-off gate into the channel; subscribe the framework's `onClick` once and offer `onPress(kind, handler)` with the per-kind claim and the held click (`presses.test.ts` in this step, over the `ask` kind alone); add the `channel`, `controls` and `tab-attention` components. Add the `olai.yml` row before `chat`. In chat: an `attention` component that needs `alertsChannel`, holds it, and mounts the circuit from its `apply`; drop `alerts`, `alert-controls`, `tab-attention`, `chat.alerts`, `alert-keys`; `Panel.tsx` stops mounting the circuit. Update `preferences_steps.ts` and `suite.testlib.ts` imports of the keys, the fence's lists, `docs/plugins/alerts.md`, `docs/index.md`, `docs/plugins/chat.md`, `docs/chat.md`, `overview.md`. `the_agent_waits_on_you.feature` and `preferences.feature` pass unchanged. Commit: `alerts: the notification channel is its own row`.
2. **The rule and the words.** `journal/src/browser/agenda/owed.ts` exports `phraseOf`; `journal/src/browser/reminders/rule.ts` and `notice.ts` with their tests; `journal/src/browser/claims.test.ts`. `NotifyClick` gains `due`; chat's press handler acts on `ask` only; the validator test covers both. Commit: `journal: the reminder rule, with no browser under it`.
3. **The circuit, the record, the row.** `reminders/said.ts` (the two preferences), `reminders/circuit.ts` (the effect, the press subscription, `go(agendaRoute)`), `reminders/ReminderRow.tsx`; the `reminders` and `reminder-controls` components on the journal's `components` record, holders for the channel, navigation and deployment. Unit tests for the circuit over fake signals (raise order: `said` first). Commit: `journal: a once-a-day reminder that the day has work on it`.
4. **The feature.** `reminders_steps.ts` with the Reminders row steps, the storage steps and the tag assertion, on the existing `@alerts` stage unchanged; `the_day_reminds_you.feature` with the eleven scenarios of §9.2. Commit: `e2e: the day reminds you, and only once`.
5. **Docs and coverage.** `docs/plugins/journal.md` (Reminders, the ruling), `docs/architecture/overview.md`, `docs/architecture/e2e-coverage.md`. Run the fast remote e2e leg, then `just ci` on the pushed head. Commit: `docs: reminders, and the pull-with-one-push ruling`.

## 13. Decided by the human, 2026-09-11

- **Ownership move ratified (§7 A):** the channel leaves chat for the tab-only `alerts` row, as written.
- **Quiet rule flipped:** the first draft had "watched at the first non-zero reading spends the day silently". That is gone. A reminder fires in a focused, watched tab too. The per-day dedupe stays exactly as written: `olai.reminders.said` recorded before raising, followed across tabs, one OS tag per day, so a reload, a reconnect, a second tab or a second task the same day never re-alert. The `@behind` tag and the `hasFocus` window steps the first draft proposed are not needed by anything and are not to be built.
- **Default on** for the Reminders row stands.
- **The chime is best-effort** (ruled on Codex's finding that a boot-time chime cannot play before a gesture, §3.5): the notification is the reminder and fires whenever the rule says; the chime plays only if audio is already unlocked and is otherwise skipped for the day, not replayed. Scenarios 1 and 9 changed accordingly; the per-day dedupe and the no-focus rule are untouched.
- **One click listener, dispatched by kind** (ruled on Codex's second finding, that the framework claims a click before its handler runs and consumes the cold-start payload at the first subscription): the alerts row subscribes the framework's `onClick` once and offers `onPress(kind, handler)`, one claimant per kind; a click whose kind is unclaimed is held, newest per kind, for the channel activation's lifetime and delivered on claim, so a cold-start `due` press opens the agenda once the journal's component is up. Chat's press handling moves to the same door with no change in behaviour. §4, §7, §8, §9 amended; everything else as ruled.

Nothing is left for the human to decide. The design is ready for Codex.
