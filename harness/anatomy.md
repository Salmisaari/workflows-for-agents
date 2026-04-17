# Harness Anatomy

## What is a harness?

The harness is everything around the LLM that turns it into an agent. The model reasons and decides; the harness handles everything else.

An agent, by definition, is an LLM interacting with tools and other sources of data. There is always a system around the LLM to facilitate that interaction.

Even web search "built into" OpenAI/Anthropic APIs is a harness — a lightweight one — sitting in front of the model, calling search via tool calling.

**Scale matters:** production harnesses are substantial — often hundreds of thousands of lines of code — while the model is hosted separately. The harness is where the system's behavior actually lives.

## Harness components

The model sits in the middle. Five components surround it:

```
                    ┌─────────────────────┐
                    │  Context Injection  │
                    │ prompts, memory,    │
                    │ skills, conversation│
                    └──────────┬──────────┘
                               │
                               ▼
   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
   │   Control   │─────▶│    Model    │─────▶│    Tools    │
   │ compaction, │      │   reasons → │      │ bash, edit, │
   │ orchestra-  │      │   decides   │      │ grep, MCPs  │
   │ tion, ralph │      │             │◀─────│             │
   │   loops     │      │             │      │  results    │
   └─────────────┘      └──────┬──────┘      └─────────────┘
                               │  ▲
                       writes  │  │ reads        ┌──────────────────┐
                               ▼  │              │ Observe & Verify │
                    ┌─────────────────┐          │ screenshots,     │
                    │     Persist     │          │ tests, logs      │
                    │ filesystem,     │          └──────────────────┘
                    │ git, progress   │
                    │ files           │
                    └─────────────────┘
```

| Component | Role | Examples |
|-----------|------|----------|
| **Context Injection** | What the model sees on each turn | system prompts, memory recall, skill metadata, conversation history |
| **Control** | When/how the model is invoked | compaction triggers, multi-agent orchestration, ralph (self-referential) loops |
| **Tools** | How decisions become side effects | bash, built-in tool calls, MCP servers; results flow back into context |
| **Observe & Verify** | Closing the loop on tool outcomes | screenshots, test results, log scraping |
| **Persist** | Durable state outside the model's context | filesystem writes, git commits, progress / scratchpad files |

## Why harness anatomy is the foundation

Treating "the layers before generation" (the harness section) as *one of several* design considerations inverts the real relationship. **The harness is the primary thing**; workflow steps are one specific harness pattern (Control + Context Injection on a sequential plan).

Leading with harness anatomy gives a workflow skill a stable spine. Workflows, agents, ralph loops, deep agents are then **harness shapes** — not separate categories.
