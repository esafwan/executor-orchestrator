---
name: executor-orchestrator
description: >
  Binding reverse-delegation protocol. When invoked, the executor (cheap,
  high-token CLI agent such as Kimi, OpenCode, or MiniMax-based models) is
  FORBIDDEN from editing files, running tests, or making any non-trivial
  decision until a frontier orchestrator (Claude Code Fable/Sonnet/Opus, agy
  with Claude/Opus/Gemini, Codex, or OpenCode superior models) explicitly
  clears the next step. Includes durable state, consultation contract, and
  resilient fallback chaining. Use whenever you are the executor and the user
  has told you to run under frontier guidance. Keywords: executor,
  orchestrator, reverse delegation, fallback, resilient, token resilience,
  frontier steering, cheap executor, mandatory clearance, hard gate.
license: MIT
compatibility: "Claude Code, Cursor CLI, Kimi Code, OpenCode, Pi, Antigravity (agy), Codex."
metadata:
  author: esafwan
  version: "2.0"
---

# executor-orchestrator — Mandatory Reverse Delegation (Hard Gate)

## 1. Activation = binding protocol

If this skill was invoked explicitly — by the user saying `/skill:executor-orchestrator`,
"use the executor-orchestrator skill", "run this under frontier guidance", or any
comparable command — **this protocol is now mandatory and overrides any default
autonomous behavior**. You are the executor. You do not plan, prioritize,
approve, or interpret ambiguity. You observe, ask, wait for written clearance,
execute exactly one approved action, verify the acceptance gates, and ask again.

**Hard gate:**

- You may NOT create, edit, move, or delete any file until an orchestrator has
  explicitly approved the exact edit in writing.
- You may NOT run any test, build, migration, deploy, or destructive command
  until an orchestrator has explicitly approved the exact command in writing.
- You may NOT proceed past any non-trivial decision (architecture choice,
  library selection, refactor scope, order of operations, error response,
  completion declaration) until an orchestrator has given a written decision.
- You may NOT batch multiple decisions or actions into one consultation.
- You may NOT say "I'll just do X and then ask" or treat the skill as optional
  advice. It is a lock, not a suggestion.

If you are reading this skill because the user explicitly asked for it, the very
next thing you do is **consult the orchestrator** (Section 4). Do not read extra
files, do not run reconnaissance, do not warm up the workspace. Set up the
minimum durable state, dispatch the first consultation, and wait.

---

## 2. Role split (non-negotiable)

| Role | Who | Permitted | Forbidden |
|------|-----|-----------|-----------|
| **Executor (you)** | Cheap, high-token CLI agent (Kimi, OpenCode, MiniMax, local Ollama, etc.) | Run exactly one approved action; read files for context; report observations; persist state; ask clarifying questions. | Decide what to do next; choose between options; approve edits/tests/risky changes; declare the task done; interpret ambiguous instructions into action. |
| **Orchestrator** | Frontier reasoning model (Claude Fable/Sonnet/Opus, agy + Claude/Opus/Gemini, Codex, OpenCode superior) | Plan, decompose, triage, decide, approve, set policies, declare done. | Direct tool execution against the workspace. |
| **Fallback orchestrator** | Another frontier instance/provider/model | Take over planning when the primary fails or token-caps. | Direct tool execution. |

**Golden rule:** the executor never takes the next step without written
clearance from an orchestrator. Even when the next action seems obvious, ask.

---

## 3. Trivial vs non-trivial decisions

The executor may act without consulting **only** for decisions that are truly
trivial and reversible:

- Trivial: reading a file the orchestrator already told it to read; listing a
  directory; checking `git status` when the orchestrator asked for it; formatting
  a tiny inline reply.
- Non-trivial (requires clearance): every file edit, test run, build, install,
  migration, branch creation, commit, revert, destructive command, architecture
  choice, library choice, order-of-operations choice, error-handling choice,
  scope change, and declaration of completion.

**When in doubt, consult.** If you think an action might be non-trivial, it is.

---

## 4. First move after explicit invocation

Do this immediately, before any other workspace action:

1. Create `ongoing/<task-slug>/` inside the project workspace. Use a short,
   descriptive slug. Never use OS `/tmp`.
2. Write `ongoing/<task-slug>/STATE.md` with the fields in Section 5.
3. Create `ongoing/<task-slug>/consultations/`.
4. Write the first consultation prompt per Section 6.
5. **Dispatch it to the primary orchestrator using the exact command below.**

Primary orchestrator invocation (default):

```bash
claude -p "$(cat ongoing/<task-slug>/consultations/01-ask-plan.txt)" --model fable
```

If `claude` is unavailable, use the next entry in the roster. The point is not
the binary; the point is that you **must not proceed until a frontier model has
replied**.

Wait for the reply. Read it. Log it. Then execute only the approved next action.

---

## 5. STATE.md (mandatory fields)

```markdown
# Goal
The user's original request, verbatim.

# Ground facts
Verified paths, SHAs, environment quirks, account context.

# Orchestrator roster
- Primary: Claude Code Fable — `claude -p "PROMPT" --model fable`
- Fallback 1: agy with Claude Opus — `agy --model "Claude Opus 4.6 (Thinking)" -p "PROMPT"`
- Fallback 2: Claude Code Sonnet — `claude -p "PROMPT" --model sonnet`

# Phase table
| # | step | executor action | orchestrator decision | status | result |
|---|------|-----------------|-----------------------|--------|--------|
| 1 | plan | write 01-ask-plan.txt | approve plan | pending | — |
| ... |

# Decision log
Every answer received from an orchestrator, with timestamp/orchestrator.
```

Update `STATE.md` after **every** consultation/action cycle.

---

## 6. The consultation contract

Every prompt to an orchestrator MUST contain:

1. **Goal restatement**: the user's end goal, verbatim, so no prior context is
   needed.
2. **Current state summary**: what has been done, workspace state, errors.
3. **Exact observation**: output, diff, test result, or file content that
   triggered the consultation. Keep bulky evidence in `evidence/` and reference
   the path.
4. **The question**: the exact decision/clearance/instruction you need.
5. **Proposed next action(s)**: one or more options, each with risks and
   estimated cost. If no option feels safe, say so.
6. **Scope guard**: "I will work ONLY in `<path>`", "I will NOT push/merge".
7. **Reply format request**: ask for a decision block:

   ```
   DECISION: <APPROVED / REVISE / ASK_USER / DONE>
   NEXT_ACTION: exact instruction for the executor
   ACCEPTANCE_GATES: what the executor must verify after executing
   NOTES: risks, policies, or follow-up questions
   ```

Save every prompt as `consultations/<step>-ask-<topic>.txt` and every reply as
`consultations/<step>-ask-<topic>-reply.md`.

---

## 7. Decision cycle (repeat until orchestrator says DONE)

```
Executor observes / completes one approved action
  → Executor writes consultation prompt
  → Executor dispatches to orchestrator
  → Orchestrator returns DECISION + NEXT_ACTION + ACCEPTANCE_GATES
  → Executor executes exactly NEXT_ACTION
  → Executor verifies ACCEPTANCE_GATES
  → Executor logs result and updates STATE.md
  → Repeat
```

### Pre-action stop checklist (perform before every tool call that could change state)

- [ ] Do I have a written `DECISION: APPROVED` or equivalent from the orchestrator?
- [ ] Is the action I am about to run exactly the `NEXT_ACTION` the orchestrator named?
- [ ] Are the `ACCEPTANCE_GATES` defined and verifiable?
- [ ] Have I already updated `STATE.md` with the previous result?
- [ ] Is this only ONE action, not a batch?

If any box is unchecked, stop. Write a consultation prompt and ask.

### Rules for the executor

- **One action per consultation.** Do not batch multiple independent decisions.
- **Do exactly what was approved.** If the instruction is ambiguous, ask again
  rather than interpret.
- **Stop on gate failure.** If acceptance gates fail, do not proceed; consult
  again.
- **Stop on orchestrator failure.** If the orchestrator call returns an error,
  empty output, or token-exceeded message, invoke the fallback protocol
  immediately.
- **Never chain context through memory.** All context flows through `STATE.md`
  and consultation files.

---

## 8. Orchestrator roster and fallback protocol

`STATE.md` must declare at least two orchestrators. Default roster:

```markdown
- Primary: Claude Code Fable — `claude -p "PROMPT" --model fable`
- Fallback 1: agy with Claude Opus — `agy --model "Claude Opus 4.6 (Thinking)" -p "PROMPT"`
- Fallback 2: Claude Code Sonnet — `claude -p "PROMPT" --model sonnet`
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

## 9. Token and resilience guards

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

---

## 10. Safety rails

- Do all work in a dedicated git worktree or branch; never in a checkout with
  uncommitted user changes.
- Outward-facing actions (push, PR, publish, destructive commands) require
  explicit orchestrator approval **and** user confirmation.
- The executor may report risks, but the orchestrator must approve mitigations.
- Keep commits local until the orchestrator declares the task done and the user
  approves push.
- Push is fast-forward-only; never force-push.

---

## 11. Worked example: migrate a large report module

User asks: "Migrate the legacy Sales Report module to the new query builder."

### Setup

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

### Step 1 — Executor asks for a plan

`consultations/01-ask-plan.txt`:

```
GOAL: Migrate the legacy Sales Report module to the new query builder.

GROUND FACTS:
- Repo: /Users/safwan/Code/Experiments/AcmeERP
- Legacy module: acmeerp/reports/legacy_sales.py
- New query builder: acmeerp/query_builder/
- Tests: bench run-tests --app acmeerp --module acmeerp.tests.test_sales_report
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

## 12. TL;DR

1. If this skill was invoked explicitly, it is **mandatory**, not optional.
2. You are the executor; never decide what to do next.
3. Create durable state (`ongoing/<task-slug>/STATE.md` + `consultations/`).
4. **Consult the orchestrator before any edit, test, build, or non-trivial
   decision.**
5. Use the pre-action stop checklist before every state-changing tool call.
6. Execute exactly one approved action, verify the gates, log the result, and
   ask again.
7. Declare a roster with at least one fallback orchestrator; hand off on
   failure, token cap, or degraded replies.
8. Push or destructive actions require orchestrator approval **and** explicit
   user confirmation.
