# Task Tracking Example

A complete end-to-end walkthrough of the openclaw-agent-workflow protocol.

**Scenario:** User asks the main agent to add a new payments feature, which requires backend API work, database migration, and frontend integration.

---

## Step 1 — User Request

```
User: Add a Stripe payments integration. We need a backend API endpoint,
      a database table for payment records, and a frontend checkout form.
```

---

## Step 2 — Main Agent: Task Planning

Main agent creates the task and updates `CURRENT_STATE.md`:

```markdown
# Current Workflow State
Last updated: 2026-03-19T10:00:00Z

## Active Tasks

| ID    | Title                       | State      | Last Event         | Owner  |
|-------|-----------------------------|------------|--------------------|--------|
| T-001 | Stripe payments integration | planned    | created            | main   |
```

Main agent reports to user:

```
[TASK T-001] Dispatched → worker agent
Plan:
- Create payments database table (migration)
- Implement /api/payments POST endpoint
- Add Stripe SDK and webhook handler
- Build frontend checkout form component
- Write integration tests
```

State in `CURRENT_STATE.md` updated to `dispatching`.

---

## Step 3 — Worker Agent: Accepted Report

Worker agent receives the task and immediately sends:

```
REPORT accepted
task_id: T-001
echo: Implement Stripe payments integration — DB migration, backend API, frontend form
plan:
  - Run database migration to create payments table
  - Install and configure stripe SDK
  - Implement POST /api/payments endpoint
  - Add Stripe webhook handler
  - Build CheckoutForm React component
  - Write integration tests
```

Main agent receives this, updates state to `in_progress`:

```markdown
| T-001 | Stripe payments integration | in_progress | accepted | worker-payments |
```

---

## Step 4 — Worker Agent: First Milestone

Worker completes the database migration:

```
REPORT milestone
task_id: T-001
step_completed: Database migration created and applied — payments table live
next_step: Installing Stripe SDK and configuring API keys
artifact: db/migrations/20260319_create_payments.sql
```

Main agent forwards to user:

```
[TASK T-001] Progress: Database migration complete
Next: Stripe SDK setup
```

`CURRENT_STATE.md` updated:

```markdown
| T-001 | Stripe payments integration | in_progress | milestone: migration done | worker-payments |
```

---

## Step 5 — Worker Agent: Blocked Report

Worker hits an issue with API key configuration:

```
REPORT blocked
task_id: T-001
blocker: Stripe API keys not present in environment. STRIPE_SECRET_KEY and
         STRIPE_WEBHOOK_SECRET are missing from .env and CI secrets.
tried:
  - Checked .env.example — keys not documented
  - Checked CI config — no Stripe secrets configured
  - Checked team docs wiki — no setup guide found
needs: Stripe API keys (test mode) added to .env and CI secrets
```

Main agent updates state to `blocked` and reports to user:

```
[TASK T-001] BLOCKED
Reason: Stripe API keys missing from environment
Tried: Checked .env, CI config, and team docs
Waiting on: STRIPE_SECRET_KEY and STRIPE_WEBHOOK_SECRET (test mode keys)
```

`CURRENT_STATE.md` updated:

```markdown
## Blocked Tasks

| ID    | Title                       | Blocker                        | Since                |
|-------|-----------------------------|--------------------------------|----------------------|
| T-001 | Stripe payments integration | Missing Stripe API keys in env | 2026-03-19T10:18:00Z |
```

---

## Step 6 — User Resolves Blocker

```
User: I've added the Stripe test keys to .env and GitHub secrets.
      STRIPE_SECRET_KEY and STRIPE_WEBHOOK_SECRET are set.
```

Main agent relays to worker. Worker resumes:

```
REPORT milestone
task_id: T-001
step_completed: Stripe SDK installed and configured — keys verified working
next_step: Implementing POST /api/payments endpoint
artifact: src/lib/stripe.ts
```

State returns to `in_progress`. `CURRENT_STATE.md` blocker row removed.

---

## Step 7 — Worker Agent: More Milestones

```
REPORT milestone
task_id: T-001
step_completed: POST /api/payments endpoint implemented with idempotency key support
next_step: Adding Stripe webhook handler
artifact: src/api/payments.ts
```

```
REPORT milestone
task_id: T-001
step_completed: Stripe webhook handler implemented and verified locally
next_step: Building CheckoutForm React component
artifact: src/api/webhooks/stripe.ts
```

```
REPORT milestone
task_id: T-001
step_completed: CheckoutForm component built with card element and error handling
next_step: Writing integration tests
artifact: src/components/CheckoutForm.tsx
```

Main agent forwards each to user with brief updates:
```
[TASK T-001] Progress: /api/payments endpoint done
[TASK T-001] Progress: Webhook handler done
[TASK T-001] Progress: CheckoutForm component done — writing tests
```

---

## Step 8 — Worker Agent: Done Report

```
REPORT done
task_id: T-001
summary: Stripe payments integration complete. Backend API endpoint with idempotency,
         webhook handler for payment confirmation events, database migration for
         payment records, and CheckoutForm frontend component with error handling.
         All integration tests passing.
artifacts:
  - db/migrations/20260319_create_payments.sql
  - src/lib/stripe.ts
  - src/api/payments.ts
  - src/api/webhooks/stripe.ts
  - src/components/CheckoutForm.tsx
  - tests/integration/payments.test.ts
success_criteria_met: true
notes: Webhook endpoint needs to be registered in Stripe dashboard before going to prod.
       See src/api/webhooks/stripe.ts line 3 for the expected URL path.
```

---

## Step 9 — Main Agent: Review

Main agent checks the artifacts, verifies tests pass, confirms success criteria:

- Database migration exists and applies cleanly
- API endpoint matches spec
- Webhook handler registered at correct path
- Frontend component renders and submits
- Tests all green

---

## Step 10 — Main Agent: Report Done to User

```
[TASK T-001] DONE
Summary: Stripe payments integration complete — API, webhook, DB migration, checkout form
Artifacts:
  - db/migrations/20260319_create_payments.sql
  - src/api/payments.ts + webhook handler
  - src/components/CheckoutForm.tsx
  - Integration tests passing

Note: Register the webhook URL in your Stripe dashboard before deploying to prod.
      Path: /api/webhooks/stripe
```

`CURRENT_STATE.md` final state:

```markdown
# Current Workflow State
Last updated: 2026-03-19T11:05:00Z

## Active Tasks
(none)

## Completed Tasks (last 5)

| ID    | Title                       | Completed At         | Artifacts                              |
|-------|-----------------------------|----------------------|----------------------------------------|
| T-001 | Stripe payments integration | 2026-03-19T11:05:00Z | payments.sql, payments.ts, +4 files    |
```

---

## Summary

This example demonstrates:

| Protocol Event | What Happened |
|---|---|
| Task created | Main agent created T-001, reported plan to user |
| `accepted` report | Worker echoed task back, state → `in_progress` |
| `milestone` reports | User saw progress at each step, never wondered "is it still running?" |
| `blocked` report | User was immediately informed of the blocker with full context |
| Blocker resolved | State returned to `in_progress` without user having to chase |
| `done` report | Full artifact list and caveats surfaced to user |
| Main agent review | State only moved to `done` after main agent verified the work |

No silent disappearances. No mystery. Every state change has evidence.
