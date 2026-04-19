---
name: scenarios
description: Use when enumerating the full range of realistic inputs a workflow must handle — malformed, adversarial, edge cases, multi-intent messages, not just the happy path. Triggers on "what inputs should we handle", "enumerate scenarios", "what could break this", "boil the ocean on inputs". Invoked by `design-workflow` as the first step after pattern selection.
---
# Scenarios

The first step of any non-trivial workflow design. The happy path is never the spec — production traffic is the long tail of weird inputs the user did not describe. This skill enumerates that long tail before any architecture is committed.

The cost of enumeration is one design session. The cost of skipping it is real edge cases shipping to production undefended.

## When to run this skill

Run this **before** pattern selection is finalized, memory architecture is committed, or any UX surface is sketched. Run it again after the first version ships, because real usage surfaces branches the design missed. Skip only for trivial pipelines with a single fixed input shape — and even then, ask once whether you're sure.

## Step 1: State the workflow's apparent scope

Ask the user, in their own words, what the workflow is supposed to do. Capture the one-sentence intent verbatim. Do not paraphrase or restructure yet — the goal is to hold the user's framing in front of the enumeration so you can notice the gap.

Example user statement: *"Build an agent for handling returns."*

That sentence is not a design. It is the prompt for one.

## Step 2: Generate the scenario tree

Prompt the user (or generate yourself if context is sufficient):

> List every realistic way this workflow's input can vary. Include happy path, edge cases, malformed inputs, adversarial inputs, multi-intent inputs, partial inputs, ambiguous inputs, and inputs that look like they belong but don't. Do not filter for plausibility — list the weird ones too.

A useful prompt to a model: *"List every way this workflow can branch, including unusual ones. Group by category. Aim for 15+ branches before stopping."*

The aim is breadth, not polish. A messy 30-item list is more useful than a clean 8-item list.

## Step 3: Cluster scenarios into a tree

Group the flat list into a tree by outcome category. The shape almost always converges on something like:

```
Happy outcome
├── variant A
├── variant B
Unhappy outcome
├── reason 1
├── reason 2
In-flight / partial
├── timing edge case
├── data edge case
Triggers other work
├── downstream system 1
├── downstream system 2
```

Returns is the canonical worked example:

```
Returns scenario tree:
├── Return accepted
│   ├── Refund (full / partial)
│   ├── Store credit / coupon
│   ├── Exchange (same SKU, different size)
│   └── Exchange (different SKU, price delta handling)
├── Return rejected
│   ├── Outside return window
│   ├── Item damaged by customer
│   ├── Final-sale item
│   └── Missing components
├── Return in limbo
│   ├── Customer never ships item
│   ├── Item lost in transit
│   └── Partial return (some items returned, some kept)
└── Return triggers other work
    ├── Inventory restock decision
    ├── Quality flag if pattern emerges
    └── Refund payment-method routing
```

Now the design conversation has substance. Now you can pick a pattern (state machine, almost certainly), define states, and reason about which branches are real.

## Step 4: Mark each branch in-scope / out-of-scope / deferred

Walk every branch. For each, classify it as one of:

- **In-scope** — v1 must handle this correctly.
- **Out-of-scope** — explicitly not the agent's job. Document who or what does handle it.
- **Deferred** — recognized but postponed. Will be added when v2 work begins.

Do not silently drop branches. A branch that doesn't appear in any of the three buckets is a branch you'll get hit by in production. The act of forcing classification is the discipline; the resulting label is the artifact.

## Step 5: Define escape hatches for deferred and out-of-scope branches

For every branch that is *not* in-scope, name what the agent does when it actually encounters that input. Three options:

- **Hand off** — route to a named human or other system.
- **Refuse** — return a structured "I can't handle this" with a reason.
- **Log and continue** — record the encounter for later review without acting.

A workflow without explicit escape hatches will silently misbehave on out-of-scope inputs. The escape hatch is what makes deferred *safe* rather than *latent failure*.

## Step 6: Schedule the re-enumeration

The enumeration is not one-time. Real production traffic will surface branches the design missed. Schedule a re-run of this skill at the first review milestone (typically two weeks after launch, or after the first 100 production runs, whichever comes first). Capture the new branches, classify them, update the escape hatches.

## Output contract

The skill produces a **Scenario Document** with five parts:

1. **Workflow intent.** The user's one-sentence statement from Step 1.
2. **Scenario tree.** The clustered tree from Step 3.
3. **Scope table.** Every branch marked in-scope / out-of-scope / deferred.
4. **Escape hatch map.** What the agent does for each non-in-scope branch.
5. **Re-enumeration schedule.** When the next pass runs.

Hand this document off to `constraints` (the next skill in the design sequence) — the in-scope branches define what the agent must be able to change, and the escape hatches are the first immutable constraints.

## Failure modes

- **Stopping too early.** If the list has fewer than 15 branches for a non-trivial workflow, push for more. The first 8 are obvious; the next 8 are where production failures live.
- **Silently dropping branches.** If a branch isn't classified in Step 4, it will hit production undefended. Force the classification.
- **No escape hatch on deferred.** A deferred branch without an escape hatch is just a hidden bug with a label.
- **Skipping the re-run.** Production surfaces branches the design missed. Without re-enumeration, coverage freezes at v1.

Back-references: `../../architecture.md` "Boil-the-ocean design heuristic" — the underlying principle and the returns example.
