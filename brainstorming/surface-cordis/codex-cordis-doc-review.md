# Codex review of the Cordis architecture guide

Follow-up to the independent proposals. Read the complete [Olai Cordis guide](https://github.com/juspay/olai/blob/master/docs/architecture/cordis.md) before reading Fable's follow-up. The original local source checkout did not yet contain this new documentation page.

The package split holds. The missing contract is finer ownership inside a plugin, and the evidence needed to justify extra composition machinery.

| Add to the plan | Why it changes an implementation |
|---|---|
| Shared implementation factories; live state per host and independent consumer | Moving a registry upstream must not turn it into a singleton. Test two app instances together. |
| Fresh registration token; explicit exclusive/multiple cardinality | Stale cleanup must not clear a replacement holding the same service object. Multi-key rejection installs nothing. |
| Child scope as well as activation owns registration | Closing one pane must end its observers even while its plugin survives. Ordinary scope ordering is not automatically activation teardown ordering. |
| Actual acquired service through `needs`, not a ready signal plus imported live variable | Export fences and dependency ownership are separate checks. A narrow broker is not `current(anyKey)`. |
| Bracket late async acquisition and dynamic UI construction | Interrupting a waiter does not cancel the underlying promise. Prevent or dispose late allocations and join post-boot acquisitions too. |
| Durable operations keep outcome policy in domain plugins | Cancellation can happen after a remote write committed. Cleanup is neither rollback nor proof of rollback. |
| Preserve surviving Wired clients and snapshot-based resubscription | Existing reconnection works with a pending gap; no uninterrupted-event guarantee. Do not add caches or blanket plugin restarts on speculation. |
| Separate navigation, layout and content responsibilities | Example board should not imply the renderer or host owns routes/history or every overlay. Concrete plugin granularity stays app-owned. |

Suggested proof additions: close one of two panes; stale release after same-object reinstallation; stop during async construction; same-process two-host isolation; optional provider absent/replaced; subscription resumes fresh data after reconnect; external result committed but acknowledgement lost.

Keep the composition snapshot a proposed integration contract. Do not present its revision/acknowledgement fields as a proven repair of Olai reconnection; first produce a failing same-name replacement or reconciliation test using existing identity, redial and reroster mechanisms.
