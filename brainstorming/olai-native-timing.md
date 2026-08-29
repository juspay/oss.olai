# Native task timing — olai tracks todo→done itself

*Ruled with the human 2026-08-29 (question tool): first-start→final-done; always visible, doing rows tick live; no `doing` phase → no `took`.*

## Storage: one NEW field, no shape changes

Marks are record FIELDS (beside `title`), not `custom` props — and each field has ONE shape (`done` always an instant, `doing`/`todo` always `true`). So no overloading: `started` is a new instant field, standing alone like `created`/`changed` do.

Every state, concretely (timestamps shortened):

```jsonc
{"id":"bake","todo":true}                                          // todo: untouched, no started
{"id":"bake","doing":true,"started":"…09:52"}                      // set_doing: stamps started
{"id":"bake","done":"…12:26","started":"…09:52"}                   // set_done: took derivable = 2h34m
{"id":"bake","cancelled":"…12:26","started":"…09:52"}              // cancelled: same shape, took = time sunk
{"id":"bake","done":"…12:26"}                                      // todo→done jump: no started, no took
{"id":"bake","doing":true,"started":"…09:52"}                      // RE-OPENED after done: started KEPT, not re-stamped
{"id":"note","title":"a bullet"}                                   // not a task: nothing here applies
```

No migration: old files lack `started` the way they lack `date`. The journal is untouched: `started` places the node on no day page (the journal reads settling instants only, as today).

## The three rules

1. **`took` = `done` − `started`.** Derived at read time, never stored — the `progress` precedent (an annotation, nothing more).
2. **Re-open keeps the first start.** `set_doing` stamps `started` only when absent; `set_done undo` + `set_doing` again re-stamps nothing. `took` is total elapsed, first start → final done. Git holds the span-by-span story.
3. **No `started`, no `took`.** A todo→done jump has no span. Never fall back to `created` — that measures the node's age, not the work.

## What a reader sees

```
read_node bake →  { ..., "started": "…09:52:00…", "done": "…12:26:49…", "took": 9284 }   // seconds, derived
```

- **done row**: `bake the bread  ⏱ 2h34m` — always visible, quiet register (the ¶-counter idiom).
- **doing row**: `bake the bread  ⏱ 47m…` — TICKING live from the stored start, pomodoro-style. The value once over the wire (the stored instant); the tick is local — the uptime chip's exact pattern (#423).

## What it retires

The orchestrator's timings step stops subtracting neighbors' `done` instants: a step's span IS its `took`. Dispatch = `set_doing` (start stamped), settle = `set_done` (span closed). The typed `took` prop in `_olai/Properties.olai` retires; the timings PR comment reads the field. And every vault gets it — a human's `house.olai` chores time themselves the same way.

## Touched surfaces

- `@olai/format`: the `started` field (validator + writer); `took` in the read projection.
- MCP `set_doing`: stamps `started` when absent (rule 2 is one `if`).
- web: the ⏱ chip on done rows; the ticker on doing rows (reuse #423's local-tick seam).
- docs/format.md: the mark table's `doing` row.
