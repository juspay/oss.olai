# Native task timing — olai tracks todo→done itself

*Ruled with the human 2026-08-29 (question tool): first-start→final-done; always visible, doing rows tick live; no `doing` phase → no `took`.*

## Storage: one field changes

Marks are record FIELDS (beside `title`), not `custom` props. `done` already stores its instant; `doing` stores a bare `true`. The change: `doing` stores its instant too.

```jsonc
// today
{"id":"bake","title":"bake the bread","doing":true}
{"id":"bake","title":"bake the bread","done":"2026-08-29T12:26:49-04:00"}

// after
{"id":"bake","title":"bake the bread","doing":"2026-08-29T09:52:00-04:00"}
{"id":"bake","title":"bake the bread","doing":"2026-08-29T09:52:00-04:00","done":"2026-08-29T12:26:49-04:00"}
```

`doing: true` in existing files stays legal — "started, instant unknown". The journal is untouched: a dated `doing` still places the node on no day page (the journal reads settling instants only, as today).

## The three rules

1. **`took` = `done` − `doing`.** Derived at read time, never stored — the `progress` precedent (an annotation, nothing more).
2. **Re-open keeps the first start.** `set_done undo` + `set_doing` again does NOT re-stamp `doing`; `took` is total elapsed, first start → final done. Git holds the span-by-span story.
3. **No `doing`, no `took`.** A todo→done jump has no span. Never fall back to `created` — that measures the node's age, not the work.

## What a reader sees

```
read_node bake →  { ..., "doing": "…09:52:00…", "done": "…12:26:49…", "took": 9284 }   // seconds, derived
```

- **done row**: `bake the bread  ⏱ 2h34m` — always visible, quiet register (the ¶-counter idiom).
- **doing row**: `bake the bread  ⏱ 47m…` — TICKING live from the stored start, pomodoro-style. The value once over the wire (the stored instant); the tick is local — the uptime chip's exact pattern (#423).

## What it retires

The orchestrator's timings step stops subtracting neighbors' `done` instants: a step's span IS its `took`. Dispatch = `set_doing` (start stamped), settle = `set_done` (span closed). The typed `took` prop in `_olai/Properties.olai` retires; the timings PR comment reads the field. And every vault gets it — a human's `house.olai` chores time themselves the same way.

## Touched surfaces

- `@olai/format`: `doing` field accepts instant (validator + writer stamps it); `took` in the read projection.
- MCP `set_doing`: stamps now (no re-stamp if `doing` already holds an instant — rule 2).
- web: the ⏱ chip on done rows; the ticker on doing rows (reuse #423's local-tick seam).
- docs/format.md: the mark table's `doing` row.
