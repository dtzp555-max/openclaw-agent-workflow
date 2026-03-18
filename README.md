# openclaw-agent-workflow

A lightweight JIRA-like workflow skill for OpenClaw agents. Prevents the three most common failure modes in multi-step agent tasks:

- **Silent disappearance** — agent stops responding with no explanation
- **Status opacity** — user has no idea what the agent is doing or how far along it is
- **Lost context on failure** — when something breaks, nobody knows what was tried or where it stopped

---

## The Problem

When an OpenClaw agent handles a complex, multi-step task by dispatching sub-agents, things can go wrong silently:

```
User: "Migrate the database schema and update all dependent services"
Agent: "On it!"
...
[10 minutes of silence]
...
Agent: "Done!" ← Did it actually finish? Was anything skipped?
```

Or worse:

```
Agent: "On it!"
...
[silence forever]
...
User: "Hello?"
```

This skill solves that by giving agents a shared protocol for state transitions, progress reporting, and timeout handling.

---

## How It Works

Every task moves through defined states with evidence requirements:

```
planned → dispatching → in_progress → reviewing → done
                              ↓
                           blocked → (resolved) → in_progress
```

Worker agents report back at key moments (`accepted`, `milestone`, `blocked`, `done`). If 10 minutes pass without a report, the main agent marks the task as `launch_failure` and alerts the user.

A live `CURRENT_STATE.md` file tracks all active, completed, blocked, and failed tasks.

---

## Installation

Copy this skill into your OpenClaw project:

```bash
cp -r openclaw-agent-workflow/ ~/.openclaw/skills/
```

Or reference it directly in your agent's skill path. Then include it in your agent configuration:

```json
{
  "skills": ["openclaw-agent-workflow"]
}
```

---

## Usage

### Starting a tracked task

When the main agent receives a multi-step task, it should:

1. Create a task entry in `CURRENT_STATE.md` with state `planned`
2. Dispatch to a worker agent with the task ID
3. Wait for the worker's `accepted` report before proceeding
4. Forward `milestone` and `blocked` reports to the user
5. Verify the `done` report before marking the task complete

### Worker agent behavior

Workers follow the reporting protocol in `SKILL.md`. Every worker must:

- Send `accepted` within 10 minutes of receiving a task
- Send `milestone` after each significant step
- Send `blocked` immediately when stuck
- Send `done` with a full summary when finished

### Example interaction

```
User: Refactor the auth module and update all callers

[TASK T-001] Dispatched → worker agent
Plan:
- Audit current auth module
- Define new interface
- Rewrite module
- Update callers
- Run tests

[TASK T-001] Progress: Auth module audit complete (12 callers found)
Next: Defining new interface

[TASK T-001] Progress: New interface defined
Next: Rewriting module

[TASK T-001] BLOCKED
Reason: CallerX uses internal auth state not exposed by new interface
Tried: Checked all public methods, reviewed git history
Waiting on: Decision — expose the state or refactor CallerX separately

User: Refactor CallerX separately

[TASK T-001] Progress: Blocker resolved, resuming module rewrite
...

[TASK T-001] DONE
Summary: Auth module refactored, 12 callers updated, CallerX refactored separately
Artifacts: src/auth/index.ts, src/auth/interface.ts, tests/auth.test.ts (+11 files)
```

---

## Files

| File | Purpose |
|---|---|
| `SKILL.md` | Full protocol definition — states, evidence rules, timeouts, report formats |
| `openclaw.plugin.json` | Skill registration metadata |
| `CURRENT_STATE.md` | Live task state (auto-managed by agents, created on first use) |
| `examples/task-tracking-example.md` | Complete worked example |

---

## See Also

- `SKILL.md` — full protocol specification
- `examples/task-tracking-example.md` — end-to-end worked example with all report types
