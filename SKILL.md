# OpenClaw Agent Workflow Skill

A lightweight task-tracking protocol for OpenClaw agents handling multi-step work. Eliminates silent failures and opaque state by requiring structured status reports at every transition.

---

## Workflow States

| State | Meaning |
|-------|---------|
| `planned` | Task accepted by main agent, not yet dispatched |
| `dispatching` | Main agent has sent task to worker; awaiting `accepted` confirmation |
| `in_progress` | Worker confirmed receipt AND has begun work (evidence required) |
| `blocked` | Worker cannot proceed; main agent must intervene |
| `reviewing` | Worker reports done; main agent is verifying output |
| `done` | Main agent has verified output and reported to user |

---

## Evidence Rules

State upgrades MUST be backed by evidence. Spawning alone does not count.

| Transition | Required Evidence |
|------------|-------------------|
| `dispatching` → `in_progress` | Worker sends `accepted` report with first action taken |
| `in_progress` → `reviewing` | Worker sends `done` report with concrete output (file path, result, diff, etc.) |
| `reviewing` → `done` | Main agent has read/verified the output artifact |
| `*` → `blocked` | Worker sends `blocked` report with specific blocker description |

**Rule:** Never write "task is in_progress" in a user-facing update unless the worker has sent an `accepted` report.

---

## Timeout Rules

- **Launch timeout:** If a worker does not send an `accepted` report within **10 minutes** of dispatch, treat the task as a launch failure.
- **Milestone timeout:** If a worker is `in_progress` and sends no update (milestone or done) for **10 minutes**, escalate to `blocked`.
- **Recovery:** On timeout, main agent must either re-dispatch or report failure to user. Never silently wait.

---

## Worker Report Protocol

Workers report back to the main agent using this structured format. All fields are required.

### On `accepted`
```
status:   accepted
summary:  <one sentence: what I understand the task to be>
evidence: <first concrete action taken, e.g. "read file X", "found 3 candidates">
risk:     <none | low | medium | high> — <brief reason if not none>
next:     <what I will do next>
```

### On `milestone`
```
status:   milestone
summary:  <what was just completed>
evidence: <artifact or output, e.g. file written, test passed, result found>
risk:     <none | low | medium | high> — <brief reason if not none>
next:     <what remains>
```

### On `blocked`
```
status:   blocked
summary:  <what I was trying to do>
evidence: <specific error, missing input, or ambiguity causing the block>
risk:     high — blocked task cannot proceed
next:     <what main agent needs to do to unblock>
```

### On `done`
```
status:   done
summary:  <what was accomplished>
evidence: <final artifact: file path, output, test result, etc.>
risk:     <none | low | medium | high> — <brief reason if not none>
next:     none
```

---

## Main Agent Report Protocol

Main agent reports to the user at these moments:
- After dispatching (transition to `dispatching`)
- After receiving `accepted` (transition to `in_progress`)
- After each `milestone` from a worker
- After verifying output (transition to `done`)
- Immediately on `blocked` or timeout

### Report Format

```
who:    <worker name or task ID>
status: <dispatching | in_progress | milestone | blocked | done>
output: <what has been produced so far, or "none yet">
next:   <what happens next, or "task complete">
```

### Timing Rules

- Do NOT wait silently. Every state change → one report to user.
- Do NOT batch multiple state changes into one delayed report.
- If nothing has changed for 5 minutes during `in_progress`, send a heartbeat to user.
