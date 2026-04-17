# Harness Openness Spectrum

Four configurations exist on a spectrum from **fully open** (you own everything) to **fully closed** (the model provider owns everything). What gets pulled behind the API is a deliberate lock-in lever.

## Stage 1 — Fully Open Harness

```
┌─────────────────────┐         ┌──────────────────┐
│      Harness        │         │ Model Provider   │
│                     │         │      API         │
│  Short term memory  │         │                  │
│  Long term memory   │ ──req──▶│      LLM         │
│  Tools              │ ◀─resp──│                  │
│  Prompt             │         │                  │
└─────────────────────┘         └──────────────────┘
```

You hold all four pillars. The model is stateless. **Full control, full portability.**

## Stage 2 — Stateful API (mildly bad)

```
┌─────────────────────┐         ┌──────────────────┐
│      Harness        │         │ Model Provider   │
│                     │         │                  │
│  Long term memory   │ ──req──▶│      LLM         │
│  Tools              │ ◀─resp──│ Short term memory│
│  Prompt             │         │                  │
└─────────────────────┘         └──────────────────┘
```

Examples: OpenAI Responses API, Anthropic server-side compaction. Short-term memory (the running conversation) lives on their server. **Cost:** swapping models forfeits resumable threads.

## Stage 3 — Black Box Harness (bad)

```
┌─────────┐  ┌──────────────────┐  ┌──────────────────┐
│ Prompt  │  │ Black Box Harness│  │ Model Provider   │
│         │  │                  │  │                  │
│ Tools   │  │ Long term memory │─▶│      LLM         │
│         │  │                  │◀─│ Short term memory│
└─────────┘  └──────────────────┘  └──────────────────┘
```

Example: Claude Agent SDK (built on closed-source Claude Code). Long-term memory artifacts exist but their shape is opaque, so they are **non-transferrable** to another harness.

## Stage 4 — Everything behind the API (worst)

```
┌─────────┐         ┌──────────────────────┐
│ Prompt  │         │ Model Provider API   │
│         │         │                      │
│ Tools   │ ──req──▶│      LLM             │
│         │ ◀─resp──│  Long term memory    │
└─────────┘         │  Short term memory   │
                    │      (Harness)       │
                    └──────────────────────┘
```

Example: Anthropic Claude Managed Agents. **Zero ownership** of memory or harness behavior. Maximum lock-in.

## Why providers push toward closed

- **Memory creates lock-in that the model alone doesn't.** Models are easy to swap (similar APIs); a memory-rich agent isn't.
- A user's memory is a **proprietary dataset** — preferences, tone, history. Whoever holds it wins the relationship.
- This is why "models will absorb the harness" is industry shorthand for "memory-related parts will move behind APIs."

## Anecdote worth remembering

A real-world case: an internal email assistant got accidentally deleted. Recreating from the same template was much worse than the original — every preference and tone detail had to be retaught from scratch. The pain proves the stickiness of accumulated memory.

## Implication for the skill

When the skill recommends architectures, it should call out which openness stage each option lands on. A workflow built on a stateful API may feel simpler today and become a migration tax tomorrow.
