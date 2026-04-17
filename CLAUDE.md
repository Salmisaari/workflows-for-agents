# CLAUDE.md

Start here. Map of the repo and rules for where things go.

## Purpose

Theory for designing agentic workflows. Will be distilled into 6 skill files (`design-workflow`, `memory-system`, `scenarios`, `constraints`, `workflow-review`, `workflow-ux`) that replace `~/.claude/skills/workflows/SKILL.md`.

## Layout

| Path | Contains |
|------|----------|
| `README.md` | Human narrative: why this exists, conclusions, status |
| `architecture.md` | Cross-layer synthesis (3-layer model, pattern catalog, design DNA, stop-at-ambiguity) |
| `persona/`, `memory/`, `skills/` | Layer theory — `principles.md` per folder |
| `tools/`, `context-injection/`, `control/`, `observe-verify/`, `persist/`, `resolvers/` | 5-component anatomy — `notes.md` per folder |
| `harness/*.md` | Runtime-shape dimensions — one file per dimension (anatomy, openness-spectrum, runtime-sizing, runtime-patterns) |

## File conventions

- `principles.md` — synthesized theory for a named layer
- `notes.md` — working notes for a single harness component
- No author attributions in body (hurts downstream skill performance)

## Routing rules

- Cross-layer insight → `architecture.md`
- Layer-scoped insight → that layer's `principles.md`
- Component-scoped insight → that component's `notes.md`
- New top-level folder → update this file first, then create
- Status / narrative updates → `README.md`, not here

## Read order

1. This file
2. `architecture.md`
3. Relevant folder based on the question
