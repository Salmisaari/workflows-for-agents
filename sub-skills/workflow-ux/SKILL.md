---
name: workflow-ux
description: Use when designing the human-in-the-loop interaction surface of an agentic workflow — latency disclosure, wait states, reminders, escalation paths, confidence-routed branching (auto / draft / clarify). Triggers on "UX for this agent", "how does the user interact", "what does the user see while waiting", "handoff to human". Pairs the token-quality tradeoff with the responsiveness axis.
---

# Workflow UX

The user-facing surface of an agentic workflow is where the token-quality bet collides with the responsiveness bet. A workflow that maximises accuracy by reading 200k tokens of context but takes 90 seconds to respond will lose to a less accurate one that responds in 4 — unless its wait state, latency disclosure, and confidence routing make the trade legible.

This skill draws those decisions before any UI is built. The output is a UX contract the implementation can be checked against.

## When to run this skill

Run after `scenarios`, `constraints`, and `memory-system`. The interaction surface depends on what the workflow does (scenarios), what it can and cannot touch (constraints), and what it remembers (memory). Re-run when adding a new channel (chat → email → API) or when latency moves more than ~2x in either direction.

## Step 1: Name the interaction surface

The surface determines almost every other UX decision. Common ones:

- **Synchronous chat** (CLI, web chat, IDE inline) — user is watching, sub-10-second turns expected.
- **Asynchronous queue** (email, ticketing, Slack DM with no presence expectation) — minutes-to-hours acceptable.
- **Background job** (scheduled, triggered, batch) — no user present until completion.
- **API consumer** (another system is the "user") — latency budget set by the consumer's SLA, not human patience.

The same workflow on different surfaces is a different UX problem. A returns agent in synchronous chat needs a thinking indicator and confidence-routed branching; the same workflow as a batch nightly job needs a per-item summary report and an exception queue.

## Step 2: Map outputs to confidence-routed branches

For every action the workflow can take, decide which of the three branches it lands in:

- **auto** — execute without asking. Use when the action is reversible *and* the model's confidence is high *and* the cost of a wrong action is low.
- **draft** — prepare the action, persist it as `pending/`, surface for one-click approval. Use when the action is reversible-but-visible (a sent message, a created PR) or the cost of being wrong is moderate.
- **clarify** — ask a targeted question before proceeding. Use when the input is genuinely ambiguous and the agent has more than one plausible interpretation.

The split lives in code, not in the prompt. The model returns `{confidence, action, reason}`; the runtime maps confidence band → branch. Document the mapping explicitly:

```
confidence ≥ 0.85  AND  action.reversibility = "low_cost"  → auto
confidence ≥ 0.60  AND  action.reversibility ≠ "irreversible" → draft
otherwise → clarify
```

A workflow that puts everything in `auto` is unsafe. A workflow that puts everything in `clarify` is annoying enough that users disable it.

## Step 3: Choose the permission-mode floor

Pick the *most permissive* mode the workflow ever runs in. Claude Code's seven-mode taxonomy is the reference:

| Mode | When to choose |
|------|----------------|
| **Plan** | The agent shouldn't execute anything until a human approves the plan. |
| **Default** | Most operations require approval. Right starting place for a new workflow. |
| **AcceptEdits** | Filesystem edits and safe shell commands auto-approve; risky ones prompt. |
| **Auto** | A classifier decides per-action; user only sees the genuinely risky ones. |
| **DontAsk** | No prompts, but deny rules still enforced. Demands tight deny lists. |
| **BypassPermissions** | Only safety-critical checks remain. Use only in trusted background jobs. |
| **Bubble** | For sub-agents — escalate to the parent's mode, do not decide locally. |

Default to `Default`. Promote to `AcceptEdits` or `Auto` only after the workflow has demonstrated competence over enough runs to justify the trust. Two governing rules sit on top of whatever mode is chosen: **deny-first evaluation** (when an action matches both an allow and a deny rule, deny wins) and **reversibility-weighted risk** (irreversible actions get tighter gates than reversible ones, regardless of mode).

## Step 4: Define what the user sees during latency

For every turn that takes longer than ~2 seconds, the wait state matters as much as the answer. Three useful states:

- **Activity indicator.** "Working…" — minimal information, useful only for sub-10-second waits.
- **Step-level disclosure.** "Reading customer profile → checking order history → drafting response." Tells the user the agent is making progress and where it is. Right for 10-60 second waits.
- **Estimated completion + cancel.** "This will take about 90 seconds. Cancel?" Right for >60 second waits and any irreversible operation.

A workflow that shows nothing during a 30-second wait will get cancelled or duplicated by an impatient user. A workflow that streams tool-call traces by default leaks implementation detail and overwhelms.

## Step 5: Define escalation and handoff paths

Every workflow has cases it can't handle — out-of-scope branches from `scenarios`, low-confidence cases from Step 2, infrastructure failures, repeat retries. For each, name where it goes:

- **Targeted clarification** — agent asks the user a specific question. The default for ambiguity.
- **Human handoff** — route to a named team or queue. The default for out-of-scope.
- **Structured refusal** — return a typed "I can't handle this" with a reason. The default for things explicitly outside the agent's authority.
- **Retry with backoff** — for transient failures, defined retry budget then escalate.

A workflow without explicit handoff paths will silently drop the inputs it can't handle. Make the path visible and the handoff target named — not "a human" but "the returns specialist queue in Zendesk."

## Step 6: Decide what the user can interrupt and undo

For long-running or batched work, users need a way out. Three controls to explicitly include or exclude:

- **Cancel in-flight.** Can the user stop the current turn? If yes, what state is rolled back?
- **Undo last action.** For side-effecting actions, is there an undo? If not, the action belongs in `draft`, not `auto`.
- **Replay from a checkpoint.** For multi-step workflows, can the user re-run from step N with edited input?

These map directly to the persistence layer (audit-before-action — `pending/` → `done/` → `failed/`). A workflow without a `pending/` queue cannot offer undo for actions that have fired; the only "undo" available is human follow-up.

## Step 7: Write the latency budget and the cost-per-turn budget

Token quality is roughly 80% of agent quality, but every token costs latency and dollars. Make the trade explicit:

- **Latency budget per turn.** Median target and a P95 cap. "Median ≤ 6s, P95 ≤ 20s" is a real constraint, not a wish.
- **Token budget per turn.** Median target and a hard cap. The cap forces compaction or sub-agent dispatch when hit.
- **Cost-per-turn ceiling.** In dollars. A workflow that would be excellent at $2/turn but is required to ship at $0.05/turn is a different workflow.

The budget shapes which compaction layers run, whether sub-agents get spawned, and which model tier is used per task class. Without numbers, the workflow optimises for nothing in particular.

## Output contract

The skill produces a **UX Contract** with seven parts:

1. **Interaction surface.** Sync chat / async queue / background job / API.
2. **Confidence-routing map.** The auto/draft/clarify rules, written as code-ready conditions.
3. **Permission-mode floor.** Most-permissive mode + deny list + reversibility rules.
4. **Latency disclosure plan.** What the user sees at each duration band.
5. **Escalation and handoff paths.** Per-failure-type routing target.
6. **Interrupt and undo controls.** What can be cancelled, undone, replayed.
7. **Latency / token / cost budget.** Numbers, not adjectives.

This contract feeds `workflow-review` later — UX is one of the five audit dimensions, and this document is what gets reviewed.

## Failure modes

- **Designing for the happy path's latency.** Measure the slow scenarios from the scenario tree, not the median. The P95 is what users remember.
- **Putting irreversible actions in `auto`.** Reversibility wins over confidence. A 99%-confident send-email-to-customer is still in `draft`.
- **No latency disclosure on long waits.** Silence is the worst answer. Even bad disclosure ("Working on it…") beats none.
- **Implicit handoff.** "Send to support" is not a handoff target. Name the queue, the system, and the on-call expectation.
- **Missing budgets.** Workflows without numeric latency / token / cost budgets drift toward "as much as it takes," and the cost surprises ship to production.

Back-references: `../../architecture.md` (token quality ≈ 80% of agent quality, stop-at-ambiguity); `../../harness/control/principles.md` (confidence-routed branching, permission modes, reversibility-weighted risk); `../../harness/persist/principles.md` (audit-before-action, the `pending/` queue that makes undo possible).
