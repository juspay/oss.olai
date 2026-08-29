# Per-page "Done: Visible" — the toggle follows the page's reading intent

*Ruled with the human 2026-08-29 (question tool): the preference is CLIENT-SIDE ONLY; the default is HIDDEN everywhere; implementor pi, tier ordinary.*

## The problem

Done-visibility is one global mode today, but what "done" means depends on the page:

- A **roadmap** page reads as "what's next" — done rows are clutter between the reader and the open work.
- The **day board** reads as "what happened" — done lanes ARE the content. With global Done:Hidden the board hollows out as the day succeeds: on 2026-08-29 the day title says "eight merges" and the page would show none of them.

One intent forced onto every page means the human flips the global toggle constantly — the flip-count is the design smell.

## The shape

- **Per page, client-side, remembered.** Each outline page carries its own Done:Visible toggle; the choice persists per-browser as view state (like scroll/zoom — pure preference, never board data, never in git, never synced). No board writes, no new props, no address-grammar change.
- **Default: hidden.** A page with no stored preference behaves exactly as today. The toggle is purely manual — no page-type special-casing, no "day-shaped" heuristic.
- **The toggle lives where Done:Hidden's control already lives**, per page instead of global — same words, same look, scoped.

## Concrete

| page | stored preference | shows done? |
|---|---|---|
| /projects/olai/roadmap/features.olai | (none) | no — today's behavior |
| /orchestrator/lanes.olai | visible (the human toggled it once) | yes — the day's merges stay on the board |
| same page, other browser | (none) | no — client-side means per-browser |

## Edges

- **Done SUBTREES stay one claim**: visibility applies to the row and its collapsed subtree together, exactly as global Done:Hidden treats them — this changes WHERE the choice lives, not what hiding means.
- **Closed day roots on lanes.olai**: with the page toggled visible, past done day-roots WOULD show — acceptable, since completed previous-day roots are trashed by convention; the reviewer should still see this case argued.
- **Zoom**: a zoomed view is the same page — the page's preference applies; zoom does not mint a second preference.
