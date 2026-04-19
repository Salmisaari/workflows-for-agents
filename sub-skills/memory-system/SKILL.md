---
name: memory-system
description: Use when designing the memory architecture for an agentic workflow — what to persist, what to compress, what to drop, where it lives. Triggers on "how should this agent remember", "design the memory system", "what memory types do we need", "context compression strategy". Walks the seven memory questions before committing to a shape.
---

# Memory System

Memory is what the harness *does*, not a feature you bolt on. The shape of memory determines portability, recoverability, and what the agent can and cannot do across turns. This skill walks seven questions whose answers, together, *are* the memory design.

The order matters: types → persistent context → skill metadata → self-modification → compaction → queryability → filesystem. Earlier answers constrain later ones, so resist the urge to skip ahead.

## When to run this skill

Run this **after** `scenarios` and `constraints` and **before** any UX or runtime implementation. Re-run whenever scope grows beyond a single session, an entity dimension is added (per-user, per-project), or compaction starts losing context the workflow needed.

## Step 1: Inventory which memory types apply

Five memory types cover almost all workflows. Walk the list and mark each as **needed**, **maybe**, or **not needed** for this workflow:

| Type | Lives in | Lifetime | Triggering question |
|------|----------|----------|---------------------|
| **Working** | Current turn's context | One turn | What does the agent see right now? |
| **Short-term** | Conversation buffer | One session | What does it remember within the session? |
| **Long-term** | External store (file, DB) | Persistent | What survives across sessions? |
| **Entity** | Per-user / per-thing record | Persistent | What does it remember *about a specific thing*? |
| **Time-sensitive** | Tagged with TTL | Until stale | What expires automatically? |

A workflow that only needs working + short-term is a stateless task agent. Add long-term for any workflow that improves over time. Add entity the moment "remember about *this user*" or "remember about *this customer*" enters scope. Add time-sensitive only when freshness is part of correctness.

## Step 2: Decide how persistent context is loaded

`CLAUDE.md`, `AGENTS.md`, and project-level instruction files are the canonical long-term memory. The decision is *how* they enter context:

- **Always-on system prompt.** Loaded on every turn. Cheap, predictable, but inflates the floor of every conversation.
- **Per-tool-call injection.** Loaded only when a relevant tool fires. Smaller floor, but the agent can act before reading.
- **Discoverable via tool.** The agent reads it on demand. Smallest floor, highest risk it never reads at all.

Default to always-on for short, identity-defining content (≤200 lines). Switch to discoverable only when the file grows large enough that the floor cost outweighs the certainty of being read.

## Step 3: Decide how skill and tool metadata is shown

Skill descriptions and tool schemas are memory too — they take context and shape behavior. Decide:

- **System prompt block.** All skills always visible. Maximum recall, maximum floor cost.
- **System message at session start.** One-time injection. Cheaper than per-turn but still always visible.
- **Just-in-time via search/router.** Agent calls a meta-tool to surface relevant skills. Smallest floor, highest latency on first use.

The break-even is around 30-40 skills. Below that, always-visible wins on simplicity. Above that, the floor cost forces a router pattern.

## Step 4: Decide whether the agent can modify its own instructions

This is a constraint question dressed as a memory question. Three positions:

- **Read-only.** Instructions are immutable to the agent. Human edits only. Safest.
- **Append-only.** Agent can add (via a `remember(...)` tool) but not edit existing entries. Self-improving without rewriting.
- **Full read-write.** Agent can edit and delete. Maximum autonomy, maximum drift risk.

Default to read-only or append-only. Full read-write requires the same human-driven revision discipline as immutable scaffolding from `constraints` — and most workflows don't actually need it.

## Step 5: Decide what survives compaction (pick pipeline layers)

When the conversation buffer fills, the harness compacts. The decision is which compaction layers to run, in what order, and what is **pinned** through every layer.

The canonical pipeline runs cheap deterministic operations before costly model-generated ones:

1. **Budget Reduction.** Per-message size caps on tool results.
2. **Snip.** Temporal trim of old turns.
3. **Microcompact.** Fine-grained, time- and cache-aware compression.
4. **Context Collapse.** Read-time projection using summaries instead of original text.
5. **Auto-compact.** Model-generated semantic compression as the last resort.

For each layer, decide: include or skip? For each piece of state, decide: pin (never compact) or eligible (compactable)?

Always pin: the system prompt, the active task description, the most recent user message, audit-log entries that have not yet been confirmed. Always eligible: tool outputs older than N turns, scratchpad reasoning, exploration that didn't lead anywhere.

A workflow that picks "one compaction strategy" is leaving capability on the table. The pipeline framing is the move.

## Step 6: Decide whether interactions are stored and queryable

Three patterns, choose one:

- **Ephemeral.** Sessions end, history vanishes. Simple, private, no longitudinal learning.
- **Logged but not queryable.** Sessions stored as plain files (audit trail) but the agent doesn't search them. Human-debuggable, no agent recall.
- **Searchable store.** SQLite + FTS5, vector store, or both. Agent invokes `search_past_sessions("topic")` when needed.

Searchable storage is what enables long-running collaborative agents — but it adds tool-call latency on every recall and a real privacy surface. Default to logged-but-not-queryable; upgrade to searchable only when "the agent needs to remember a past conversation" becomes a real, recurring need.

## Step 7: Decide how the filesystem is represented

The filesystem is the cleanest long-term memory available — `git diff`-able, human-editable, portable. The decision is *how much of it the agent sees*:

- **No filesystem.** Pure conversational agent. All state in memory or external API.
- **Sandbox directory.** Agent sees one isolated directory. Default for most workflows.
- **Project root.** Agent sees the working directory and can read/write within it. Default for code agents.
- **Whole disk with allow/deny lists.** Agent has filesystem access governed by explicit rules.

Couple this with the `constraints` decision on what's mutable vs. immutable. The filesystem boundary and the constraint boundary should match — if the agent can see a file but cannot edit it, that's a runtime hook, not a memory choice.

## Step 8: Choose the provider, last

Only after the seven questions are answered does provider selection matter. The provider is an implementation detail behind an interface. The recurring rule: **if you can't `git diff` your memory, you don't really own it.** Plain files first, SQLite second, managed services only when the workflow's recall pattern genuinely needs them.

## Output contract

The skill produces a **Memory Architecture Document** with eight parts:

1. **Memory types in use.** The needed/maybe/not-needed table from Step 1.
2. **Persistent context loading.** How `CLAUDE.md`-class files enter context.
3. **Skill / tool metadata exposure.** Always-on vs. on-demand, with rationale.
4. **Self-modification policy.** Read-only / append-only / read-write, with the channel for human-driven changes.
5. **Compaction pipeline.** Which layers run, in what order, and what is pinned.
6. **Interaction storage.** Ephemeral / logged / searchable, with retention policy.
7. **Filesystem exposure.** Scope of the agent's filesystem view.
8. **Provider choice.** With one-line justification of why this backend over plain files.

Hand this document to `workflow-ux` (next in the design sequence) — UX decisions about clarification and recall lean directly on what the memory system can and cannot answer.

## Failure modes

- **Skipping persistent context loading.** A workflow that doesn't decide how `CLAUDE.md` is loaded will load it inconsistently across runs, and behavior will drift for reasons that look like model variance.
- **Picking one compaction strategy.** Choosing "we'll just summarize" or "we'll just truncate" misses that compaction is a pipeline. Cheap operations should always run before costly ones.
- **Searchable storage by default.** Adding a vector store before the workflow needs it imports privacy surface and operational complexity for capability nobody is using yet.
- **Mismatch between filesystem view and constraint boundary.** If the agent can see a file it cannot legally modify, expect repeated failed-edit loops. Align the two boundaries.
- **Provider lock-in.** Choosing a managed memory service before answering the seven questions hides the design decisions behind an API and makes future migration painful.

Back-references: `../../memory/principles.md` (memory IS the harness, types, compaction pipeline, search-augmented, providers); `../../harness/persist/principles.md` (files as the cleanest long-term memory).
