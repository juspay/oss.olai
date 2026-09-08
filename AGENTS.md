# Orchestrator entry point

This vault is the orchestrator's durable memory. Before reading any `.olai` file, discover the olai MCP tools and their schemas. Read and write `.olai` content only through those tools; never use shell/file reads or raw edits. Git history is the sole read exception.

Boot in order:

1. Read the `supervision` subtree in `orchestrator/orchestrator.olai` through MCP: current policy, roster, harness facts and standing memory.
2. Read the `lanes` conventions node in `orchestrator/lanes.olai`.
3. Find active lanes with MCP search and read their owning step subtrees, following mirror targets. Load completed lanes only for a relevant historical question.
4. Consult product documentation for the action being taken. Chat and wake controls: `~/code/olai/docs/chat.md`. Addresses: the address material in `~/code/olai/docs/format.md`.

Repeat this chain after compaction. The subtree is memory; a conversation summary is only a navigation aid. Boot-only requests do not dispatch work. Before fleet supervision, have the human select the day board in this conversation's wake pickers; only the human can arm them.

Workflow policy lives on the board. Routine memory updates are authorized; policy changes require explicit approval. Native children may handle bounded investigations and review assistance only under `orch-native-delegation`; durable implementation lanes remain in kolu.
