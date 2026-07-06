# executor-orchestrator

> **Resilient reverse delegation.** Let a cheap, high-token executor (Kimi, OpenCode, MiniMax, local Ollama) do the work while a frontier model (Claude Opus/Fable/Sonnet, agy + Opus, Codex) makes every decision. If the orchestrator maxes out tokens or fails, automatically fall back to another one.

---

## The problem this solves

When we let larger frontier models drive smaller executors like Kimi, we often hit a frustrating wall: **the orchestrator runs out of tokens first**. The cheap executor still has plenty of budget, but the smart model that was steering it goes silent mid-task. The whole job dies.

This pattern flips the relationship but keeps the safety:

- The **executor** is the main runtime agent. It holds the tools, edits files, runs tests, and persists state.
- The **executor never decides what to do next**. It asks the orchestrator for every plan, prioritization, and clearance.
- The **orchestrator** is a frontier reasoning model used only for decisions, not for bulk execution.
- If the primary orchestrator fails or hits its limit, a **fallback orchestrator** takes over from the durable state file.

Result: you can confidently pair a low-cost, high-token executor with a powerful but token-limited orchestrator, knowing the task can survive a handoff.

---

## How it differs from orchestrator-executor

This repo is the **mirror** of [`esafwan/orchestrator-executor`](https://github.com/esafwan/orchestrator-executor).

| Pattern | Who decides | Who executes | Best for |
|--------|-------------|--------------|----------|
| **orchestrator-executor** | Frontier model plans and steers | Cheap executor works autonomously in phases | Large reviews, migrations, audits where the planner wants to delegate bulk work |
| **executor-orchestrator** | Cheap executor holds runtime but asks frontier model for every decision | Frontier model decides; cheap executor acts one step at a time | Risky or ambiguous tasks where you want frontier judgment on every move, plus token-resilient execution |

Use them together: one for "give Kimi a chunk and review later", the other for "ask Claude before every edit".

---

## Supported agents

### Executors (high token, low cost)

| Agent | Example invocation | Notes |
|-------|--------------------|-------|
| **Kimi Code CLI** | `kimi -p "PROMPT"` | Large context, good for repo-heavy loops |
| **OpenCode** | `opencode run "PROMPT"` | Multi-provider; works well with MiniMax/Ollama |
| **MiniMax / OpenCode Go** | via `opencode` provider | Very cheap, very high token budgets |
| **Local Ollama** | via OpenCode or Pi | Private, unlimited tokens, weaker reasoning |

### Orchestrators (frontier reasoning)

| Agent | Example invocation | Notes |
|-------|--------------------|-------|
| **Claude Code** | `claude -p "PROMPT" --model opus/fable/sonnet` | Strong general coding and judgment |
| **Antigravity (`agy`)** | `agy --model "Claude Opus 4.6 (Thinking)" -p "PROMPT"` | Can run Claude Opus through the Google SDK |
| **Codex** | `codex exec "PROMPT"` | GPT-Codex reasoning |
| **OpenCode superior models** | `opencode run "PROMPT"` with high-end provider | Provider-dependent |

**Current focus:** Claude Code and `agy` are fully supported in the skill. Other orchestrators work as long as they expose a non-interactive prompt mode.

---

## Quick start

1. Copy `SKILL.md` into your agent's skills directory, or symlink it:

   ```bash
   ln -s /path/to/executor-orchestrator/SKILL.md \
         ~/.kimi-code/skills/executor-orchestrator/SKILL.md
   ```

2. When the executor receives a large or risky task, create a durable state dir:

   ```bash
   mkdir -p ongoing/<task-slug>/consultations
   ```

3. Write `STATE.md` with the goal, ground facts, orchestrator roster, and phase table.

4. For every step:
   - The executor writes a consultation prompt.
   - The executor dispatches it to the primary orchestrator.
   - The executor runs exactly the returned `NEXT_ACTION`.
   - The executor verifies the `ACCEPTANCE_GATES`.
   - The executor updates `STATE.md` and asks again.

5. If the orchestrator fails, run the **fallback protocol** in `SKILL.md`:
   - Write a handoff file with `STATE.md` and recent decisions.
   - Dispatch to the next orchestrator in the roster.
   - Continue from the durable state.

---

## Why this is more resilient

- **Token isolation.** The frontier model spends tokens only on decisions, not on reading large files or writing diffs. The cheap executor spends tokens on execution, where it is strongest.
- **Budget diversity.** If Claude Opus hits its output cap, agy Claude Opus or Claude Fable may still have budget on a different account/provider.
- **State survives sessions.** `STATE.md` plus the `consultations/` log is the resume point. A fresh executor or orchestrator can pick up mid-task.
- **No single point of failure.** The executor monitors the orchestrator's health and switches before a hard failure when possible.

---

## Repository layout

```
executor-orchestrator/
├── SKILL.md      # The agent skill file (frontmatter + full pattern)
└── README.md     # This file
```

`SKILL.md` is the source of truth. Install it where your agent can load it.

---

## Future plans

- Add sample `STATE.md` templates for common task types.
- Add shell helpers for dispatch and handoff.
- Document orchestrator-specific prompt tuning.
- Provide worked examples for migrations, security audits, and merge-readiness passes.

---

## License

MIT

---

## See also

- [`esafwan/orchestrator-executor`](https://github.com/esafwan/orchestrator-executor) — the forward pattern: frontier model orchestrates a cheap executor in autonomous phases.
- [`esafwan/swarm-plus-plus`](https://github.com/esafwan/swarm-plus-plus) — resilient multi-agent orchestration and agent ladders.
