# Problem: when a lane needs the human's word, nothing reaches them

*Stated 2026-08-29; solution deliberately left open — the human is thinking on it.*

## The problem

The pipeline regularly stops at gates only the human can clear: a deferral to rule, a draft to
approve, a design decision to review, a merge word to give. When that happens today, the
orchestrator asks in chat prose and files an entry in the Inbox — and neither reaches a human who
is not looking at the screen. olai's alert machinery (chime, system notification, the badge that
stays until you look) fires only when an agent's turn stops on a *form*; prose rings nothing, and
Inbox entries filed as bare bullets don't even count on the badge.

Real case: olai#432 sat fully green for half an hour on 2026-08-29, waiting on a one-word ruling
the human didn't know was owed.

## What a solution must do

Make "the orchestrator owes you N words" reach the human — wherever they are, without them
polling chat — and make each owed word findable with its context when they arrive.

## Solution: unknown, on purpose

The human's early instinct: some sort of **live-property question thing** — plausibly the
live-properties seam (odu-in-olai) growing a face for "a question is waiting here", so a gate is a
living thing on the board rather than a sentence in a chat log. Not designed yet.

Rejected as insufficient on their word (2026-08-29): the quick pair of "ask via the question-tool
form + mark Inbox entries todo" — the interim is instead simply using the question tool where it
already fits.
