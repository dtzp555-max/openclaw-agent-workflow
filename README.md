# openclaw-agent-workflow

A lightweight task-tracking skill for OpenClaw agents. Gives multi-step agent work a clear, observable lifecycle — no more silent failures, no more mystery state.

---

## The Problem

When an OpenClaw agent dispatches sub-agents to handle complex tasks, three failure modes appear regularly:

**1. Silent disappearance**
```
User: "Deploy the new service and update the config"
Main agent: "On it!"
... [10 minutes of silence] ...
User: "Hello?"
```

**2. Status opacity**
```
Main agent: "Working on it..."
User: "How far along are you?"
Main agent: "Still working..."
```

**3. Lost context on failure**
```
Main agent: "Something went wrong."
User: "What was tried? Where did it stop? What do I need to fix?"
Main agent: [no useful answer]
```

This skill solves all three by requiring agents to report at every meaningful state transition, with evidence.

---

## Installation

**Step 1:** Copy the skill into your OpenClaw skills directory:

```bash
cp -r openclaw-agent-workflow/ ~/.openclaw/skills/
```

**Step 2:** Reference it in your agent config:

```json
{
  "skills": ["agent-workflow"]
}
```

That's it. The skill is loaded automatically when the agent starts.

---

## Usage Scenarios

### Scenario 1: Multi-service refactor

```
User: Refactor the auth module and update all 12 callers

who:    worker-a
status: dispatching
output: none yet
next:   worker-a to confirm acceptance

who:    worker-a
status: in_progress
output: read auth/index.ts, identified 12 callers
next:   rewriting module core

who:    worker-a
status: milestone
output: auth/index.ts rewritten, 9/12 callers updated
next:   3 callers remain (PaymentService, AdminAPI, LegacyBridge)

who:    worker-a
status: done
output: all 12 callers updated, tests passing (auth.test.ts)
next:   task complete
```

### Scenario 2: Worker gets blocked

```
User: Migrate the database schema

who:    worker-b
status: in_progress
output: migration script written (migrations/0042_schema.sql)
next:   running migration

who:    worker-b
status: blocked
output: migration/0042_schema.sql exists, but prod DB is read-only
next:   need DB write credentials or manual approval to proceed
```

### Scenario 3: Launch failure (timeout)

```
User: Run the full integration test suite

who:    worker-c
status: dispatching
output: none yet
next:   awaiting accepted report

[10 minutes pass with no response]

who:    worker-c
status: blocked
output: no accepted report received within 10 minutes
next:   retry dispatch or cancel — user decision required
```

---

## State Transition Diagram

```
                    +----------+
                    | planned  |
                    +----------+
                         |
                    (dispatch)
                         |
                         v
                  +-------------+
                  | dispatching |
                  +-------------+
                         |
               (worker: accepted)
                         |
                         v
                  +------------+
          +-----> | in_progress|
          |       +------------+
          |            |  |
          |    (worker: |  | (worker:
          |  milestone) |  |   done)
          |             |  |
          |   [report   |  v
          |   to user]  | reviewing |
          |             +----------+
          |                  |
          |         (main: verified)
          |                  |
          |                  v
          |              +------+
          |              | done |
          |              +------+
          |
          | (blocked → resolved)
          |
     +--------+
     | blocked|
     +--------+
          |
   (user unblocks)
          |
     [back to in_progress]
```

---

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Full protocol: states, evidence rules, timeouts, report formats |
| `openclaw.plugin.json` | Skill registration metadata |
| `examples/task-tracking-example.md` | End-to-end worked example |
