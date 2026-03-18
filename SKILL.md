# OpenClaw Agent Workflow Skill

This skill enforces a lightweight JIRA-like workflow for multi-step agent tasks, preventing silent failures and status opacity.

---

## Workflow States

| State | Meaning |
|---|---|
| `planned` | Task defined, not yet dispatched |
| `dispatching` | Main agent sending task to worker agent |
| `in_progress` | Worker agent actively executing |
| `blocked` | Worker cannot proceed without external input |
| `reviewing` | Worker done, main agent verifying output |
| `done` | Task verified and complete |

### State Transition Rules

```
planned → dispatching  : main agent sends task
dispatching → in_progress : worker sends "accepted" report
in_progress → blocked  : worker hits unresolvable blocker
in_progress → reviewing : worker sends "done" report
blocked → in_progress  : blocker resolved
reviewing → done        : main agent verifies and closes
reviewing → in_progress : main agent rejects, worker resumes
```

---

## Evidence Rules

A state upgrade is only valid when the following evidence exists:

| Transition | Required Evidence |
|---|---|
| `dispatching → in_progress` | Worker emits `accepted` report with task echo |
| `in_progress → reviewing` | Worker emits `done` report with artifact reference or summary |
| `blocked → in_progress` | Blocker resolved message from worker or user |
| `reviewing → done` | Main agent confirms output matches success criteria |

**No state may be skipped.** A task cannot go from `in_progress` directly to `done` without a `reviewing` step.

---

## Timeout Rules

| Condition | Action |
|---|---|
| Worker silent for **10 minutes** after dispatch | Mark state as `launch_failure`, report to user |
| Worker silent for **10 minutes** during `in_progress` | Escalate to `blocked`, report to user |
| Blocker unresolved for **30 minutes** | Report stale block to user, ask for guidance |

When a timeout fires, main agent must:
1. Update `CURRENT_STATE.md` with the timeout event
2. Notify the user with the last known state and elapsed time
3. Await user instruction before retrying or canceling

---

## Worker Agent Reporting Protocol

Worker agents MUST report back at these moments:

### 1. `accepted` — Task received and understood
```
REPORT accepted
task_id: <id>
echo: <one-line restatement of the task>
plan: <bullet list of planned steps>
```

### 2. `milestone` — Significant progress checkpoint
```
REPORT milestone
task_id: <id>
step_completed: <what just finished>
next_step: <what is starting now>
artifact: <file path or output reference, if any>
```

### 3. `blocked` — Cannot proceed
```
REPORT blocked
task_id: <id>
blocker: <clear description of what is blocking>
tried: <what was already attempted>
needs: <what is needed to unblock>
```

### 4. `done` — Task complete
```
REPORT done
task_id: <id>
summary: <what was accomplished>
artifacts: <list of files created/modified>
success_criteria_met: true|false
notes: <any caveats or follow-up items>
```

Workers that do not send any report within 10 minutes of accepting a task are considered to have silently failed.

---

## Main Agent User-Reporting Protocol

Main agent reports to the user at these moments:

### On dispatch
```
[TASK <id>] Dispatched → worker agent
Plan: <summary of steps>
```

### On milestone (forwarded from worker)
```
[TASK <id>] Progress: <step_completed>
Next: <next_step>
```

### On block
```
[TASK <id>] BLOCKED
Reason: <blocker>
Tried: <what was attempted>
Waiting on: <what is needed>
```

### On completion
```
[TASK <id>] DONE
Summary: <what was accomplished>
Artifacts: <list>
```

### On timeout / launch failure
```
[TASK <id>] TIMEOUT — no response for 10 minutes
Last state: <state>
Action needed: retry / cancel / investigate
```

Main agent should NOT silently proceed to the next task after a failure. Always surface the failure to the user.

---

## CURRENT_STATE.md Update Triggers

Main agent must update `CURRENT_STATE.md` whenever:

1. A new task is created (`planned`)
2. A task is dispatched (`dispatching`)
3. A worker report is received (any report type)
4. A timeout fires
5. A task reaches `done`

### CURRENT_STATE.md Format

```markdown
# Current Workflow State
Last updated: <ISO timestamp>

## Active Tasks

| ID | Title | State | Last Event | Owner |
|---|---|---|---|---|
| T-001 | <title> | in_progress | milestone: step 2/4 | worker-a |

## Completed Tasks (last 5)

| ID | Title | Completed At | Artifacts |
|---|---|---|---|
| T-000 | <title> | <timestamp> | <paths> |

## Blocked Tasks

| ID | Title | Blocker | Since |
|---|---|---|---|

## Failed / Timed Out Tasks

| ID | Title | Failure Reason | Since |
|---|---|---|---|
```
