---
name: executor-orchestrator
description: >
  Reverse delegation pattern: the executor (cheap, high-token CLI agent such as
  Kimi, OpenCode, or MiniMax-based models) does the work but NEVER decides what
  to do next. Every decision, plan, and clearance is requested from a frontier
  orchestrator (Claude Code Sonnet/Fable/Opus, agy with Claude/Opus/Gemini,
  Codex, or OpenCode superior models). Includes durable state, a consultation
  contract, and resilient fallback chaining so the task survives when the
  primary orchestrator hits token limits, rate limits, or failures. Use whenever
  you are the executor and want to let a stronger model steer while you provide
  the tokens and tool calls. Keywords: executor, orchestrator, reverse
  delegation, fallback, resilient, token resilience, frontier steering, cheap
  executor, consultation, clearance.
license: MIT
compatibility: "Claude Code, Cursor CLI, Kimi Code, OpenCode, Pi, Antigravity (agy), Codex."
metadata:
  author: esafwan
  version: "1.0"
---

# executor-orchestrator — Resilient Reverse Delegation

Use this skill when **you are the executor**: a cheap, scriptable, high-token
CLI agent that can run tools and edit files, but you must **not** decide what to
do next. A stronger frontier model (the orchestrator) makes every plan,
prioritization, and go/no-go decision. You ask, wait for clearance, then act
exactly once. If the primary orchestrator fails or runs out of tokens, you
gracefully fall back to a secondary orchestrator and continue from durable
state.

This is the mirror of `orchestrator-executor`. In that pattern the orchestrator
plans and the executor works autonomously. In this pattern the executor holds
the runtime but is deliberately **decision-passive**; the orchestrator is the
brain, and the executor is the hands.

---

## When to use

Engage this pattern when **any** of these is true:

- You are a high-token, low-cost executor (Kimi, OpenCode Go/MiniMax, local
  Ollama, etc.) and want a frontier model (Claude Opus/Fable/Sonnet, agy with
  Opus/Gemini, Codex) to make the hard calls.
- The task is too large or too risky to plan yourself, and you would rather
  spend tokens on tool execution than on reasoning about what to do next.
- You want **token resilience**: if the orchestrator hits its output cap, you
  swap to a backup orchestrator and resume instead of failing the whole task.
- You are asked to "let Claude decide", "ask the orchestrator", "run this under
  frontier guidance", or "keep executing but check every step".

If the task is small enough to finish directly and safely, do **not** add this
overhead — just do it.

---

## Role split (hard rule)

| Role | Who | Permitted | Forbidden |
|------|-----|-----------|-----------|
| **Executor (you)** | Cheap, high-token CLI agent (Kimi, OpenCode, MiniMax, etc.) | Run exactly one approved action; read files; edit files; run tests; report observations; persist state. | Decide what to do next, prioritize, choose strategies, approve risky changes, or skip steps. |
| **Orchestrator** | Frontier model with strong reasoning (Claude Sonnet/Fable/Opus, agy + Claude/Opus/Gemini, Codex, OpenCode superior) | Plan, decompose, triage, decide, approve, set policies, declare done. | Direct tool execution against the workspace. |
| **Fallback orchestrator** | Another frontier instance/provider/model | Take over planning when the primary fails/token-caps. | Direct tool execution. |

**Golden rule:** the executor never takes the next step without written
clearance from an orchestrator. Even when the next action seems obvious, ask.

---

## Setup (before the first action)

1. Create a durable state directory inside the project workspace, e.g.
   `ongoing/<task-slug>/`. Never use OS `/tmp`; it must survive session death.
2. Write `STATE.md` inside it with:
   - **Goal**: the user's original request, verbatim.
   - **Ground facts**: verified paths, SHAs, environment quirks, account
     context.
   - **Orchestrator roster**: primary and fallback orchestrators with exact
     invocation commands and model strings.
   - **Phase table**: `# | step | executor action | orchestrator decision | status | result`.
   - **Decision log**: every answer received from an orchestrator.
3. Create a `consultations/` subdirectory. Save every prompt sent to an
   orchestrator and every reply received (auditable, resumable).
4. Update `STATE.md` after **every** consultation/action cycle.

---

## The consultation contract (every prompt to an orchestrator MUST contain)

1. **Goal restatement**: the user's end goal, so the orchestrator does not
   need prior context.
2. **Current state summary**: what has been done, what the workspace looks
   like, and any errors/failures.
3. **The exact observation**: the output, diff, test result, or file content
   that triggered this consultation.
4. **The question**: what decision/clearance/instruction you need.
5. **Proposed next action(s)**: one or more options, each with risks and
   estimated cost. If no option feels safe, say so.
6. **Scope guard**: "I will work ONLY in `<path>`", "I will NOT push/merge".
7. **Reply format request**: ask the orchestrator to reply with a decision
   block (`DECISION:`, `NEXT_ACTION:`, `ACCEPTANCE_GATES:`, `NOTES:`).

---

## Decision cycle (repeat until orchestrator says DONE)

```
Executor observes / completes one action
  → Executor writes consultation prompt
  → Executor dispatches to orchestrator
  → Orchestrator returns DECISION + NEXT_ACTION + ACCEPTANCE_GATES
  → Executor executes exactly NEXT_ACTION
  → Executor verifies ACCEPTANCE_GATES
  → Executor logs result and updates STATE.md
  → Repeat
```

Rules for the executor:

- **One action per consultation.** Do not batch multiple independent decisions.
- **Do exactly what was approved.** If the instruction is ambiguous, ask again
  rather than interpret.
- **Stop on gate failure.** If acceptance gates fail, do not proceed; consult
  again.
- **Stop on orchestrator failure.** If the orchestrator call returns an error,
  empty output, or token-exceeded message, invoke the fallback protocol
  immediately.

---

## Orchestrator roster and fallback protocol

`STATE.md` must declare at least two orchestrators. Example roster:

```markdown
# Orchestrator roster
- **Primary**: Claude Code — `claude -p "PROMPT" --model fable`
- **Fallback 1**: agy with Claude Opus — `agy --model "Claude Opus 4.6 (Thinking)" -p "PROMPT"`
- **Fallback 2**: Claude Code Sonnet — `claude -p "PROMPT" --model sonnet`
```

**When to fall back:**

- Non-zero exit code from the orchestrator CLI.
- Output contains token-limit phrases (`max tokens`, `context length`, `output
  limit`, `rate limit`, `insufficient_quota`).
- Output is empty, truncated, or nonsensical.
- Orchestrator replies "I cannot continue" or refuses the decision role.
- Three consecutive consultations produce decisions that fail acceptance gates
  when executed (possible model degradation).

**How to fall back:**

1. Write a handoff prompt to `consultations/HANDOFF-<n>.md` containing:
   - The original goal.
   - The current `STATE.md` contents.
   - The last 3–5 decisions/results.
   - The reason for the fallback.
   - "You are now the active orchestrator. Reply with DECISION + NEXT_ACTION."
2. Dispatch to the next orchestrator in the roster.
3. Do not retry the failed orchestrator for the current task.
4. Update `STATE.md` with the active orchestrator and the handoff reason.

---

## Token and resilience guards

- **Executor token budget is for execution, not planning.** Keep consultation
  prompts focused; append bulky evidence as file paths, not pasted content.
- **Prefer file-backed evidence.** Instead of pasting a large diff into the
  orchestrator prompt, write it to `consultations/<step>-evidence.md` and ask
  the orchestrator to read it.
- **Monitor orchestrator response length.** If replies are growing shorter or
  start truncating mid-sentence, prepare a handoff before the cap hits.
- **Durable state is the resume point.** A fresh executor session with zero
  context must be able to continue by reading `STATE.md` and the latest
  consultation file.
- **Never chain context through executor memory.** All context flows through
  files.

---

## Safety rails

- Do all work in a dedicated git worktree or branch; never in a checkout with
  uncommitted user changes.
- Outward-facing actions (push, PR, publish, destructive commands) require
  explicit orchestrator approval **and** user confirmation.
- The executor may report risks, but the orchestrator must approve mitigations.
- Keep commits local until the orchestrator declares the task done and the user
  approves push.
- Push is fast-forward-only; never force-push.

---

## Worked example: migrate a large report module

User asks: "Migrate the legacy Sales Report module to the new query builder."

**Setup:**

```
ongoing/sales-report-migration/
├── STATE.md
├── consultations/
│   ├── 01-ask-plan.txt
│   ├── 01-ask-plan-reply.md
│   ├── 02-ask-first-file.txt
│   ├── 02-ask-first-file-reply.md
│   └── ...
└── evidence/
    ├── current-report-files.md
    └── test-failures.md
```

`STATE.md`:

```markdown
# Goal
Migrate the legacy Sales Report module to the new query builder.

# Ground facts
- Repo: /Users/safwan/Code/Experiments/AcmeERP
- Legacy module: `acmeerp/reports/legacy_sales.py`
- New query builder: `acmeerp/query_builder/`
- Tests: `bench run-tests --app acmeerp --module acmeerp.tests.test_sales_report`
- Executor: Kimi Code CLI (`kimi -p`).

# Orchestrator roster
- Primary: Claude Code Fable — `claude -p "PROMPT" --model fable`
- Fallback: agy Claude Opus — `agy --model "Claude Opus 4.6 (Thinking)" -p "PROMPT"`

# Phase table
| # | step | executor action | orchestrator decision | status | result |
|---|------|-----------------|-----------------------|--------|--------|
| 1 | plan | write 01-ask-plan.txt | approve migration plan | pending | — |
| 2 | map | write 02-ask-first-file.txt | pick first file to migrate | pending | — |

# Decision log
(empty)
```

**Step 1 — Executor asks for a plan.** `consultations/01-ask-plan.txt`:

```
GOAL: Migrate the legacy Sales Report module to the new query builder.

GROUND FACTS:
- Repo: /Users/safwan/Code/Experiments/AcmeERP
- Legacy module: acmeerp/reports/legacy_sales.py
- New query builder: acmeerp/query_builder/
- Executor: Kimi Code CLI.
- I will work ONLY in the repo above and will NOT push.

CURRENT STATE: nothing done yet.

OBSERVATION: I have not inspected the code yet. I can run reconnaissance if
you approve, or I can provide a directory listing first.

QUESTION: Should I run reconnaissance to list report files and their
dependencies, then return for a prioritized migration plan?

PROPOSED NEXT ACTION:
A. Run reconnaissance now and report back for a plan.
B. Provide a directory listing first, then decide.

Please reply with:
DECISION: A or B
NEXT_ACTION: exact command or instruction for me to execute
ACCEPTANCE_GATES: what I must verify after executing
NOTES: any risks or policies
```

Dispatch to primary orchestrator:

```bash
claude -p "$(cat consultations/01-ask-plan.txt)" --model fable
```

The orchestrator replies with `DECISION: A`, `NEXT_ACTION: ...`, and gates. The
executor runs reconnaissance, writes the result to `evidence/recon.md`, and
returns for step 2. This continues until the orchestrator replies
`DECISION: DONE`.

If Claude Fable's reply is truncated at step 5, the executor writes a handoff
prompt and dispatches to agy Claude Opus, resuming from `STATE.md`.

---

## TL;DR

1. You are the executor; never decide what to do next.
2. Create a durable state dir with `STATE.md` and `consultations/`.
3. Ask the orchestrator for clearance, execute exactly one action, verify gates,
   log the result, and ask again.
4. Declare a roster with at least one fallback orchestrator; hand off on
   failure, token cap, or degraded replies.
5. Keep bulky evidence in files; pass context through `STATE.md`, not memory.
6. Push or destructive actions require orchestrator approval **and** explicit
   user confirmation.
