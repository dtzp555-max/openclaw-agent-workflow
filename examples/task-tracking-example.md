# Task Tracking Example: End-to-End Walkthrough

This example traces a complete task lifecycle from user request to final delivery. Every state transition includes the actual report that triggered it.

**Scenario:** "Audit the API rate-limiting code, fix any bugs you find, and add tests."

---

## 1. User Request → `planned` → `dispatching`

User submits the request. Main agent registers the task and dispatches to worker-a.

**Main agent → User:**
```
who:    T-007 / worker-a
status: dispatching
output: none yet
next:   worker-a to confirm acceptance
```

---

## 2. Worker Receives Task → `in_progress`

Worker-a reads the relevant files and sends `accepted` immediately.

**Worker-a → Main agent:**
```
status:   accepted
summary:  audit src/middleware/rateLimit.ts for bugs, fix issues, add regression tests
evidence: read rateLimit.ts (312 lines) and tests/rateLimit.test.ts (88 lines)
risk:     none — read-only phase, no changes yet
next:     scan for off-by-one errors, clock skew, missing edge cases
```

Main agent transitions to `in_progress` and reports:

**Main agent → User:**
```
who:    T-007 / worker-a
status: in_progress
output: rateLimit.ts (312 lines) and test file read
next:   auditing for bug patterns
```

---

## 3. Worker Finds Issues → `milestone`

Audit complete. Worker-a reports findings.

**Worker-a → Main agent:**
```
status:   milestone
summary:  audit complete — 2 bugs found
evidence:
  - line 87: Date.now() used for window reset → vulnerable to clock skew
  - line 134: counter not reset on IP ban lift → legit users permanently limited after unban
risk:     medium — correctness bugs in production, not data-loss severity
next:     fix line 87 (clock skew), then line 134 (counter reset), then add tests
```

**Main agent → User:**
```
who:    T-007 / worker-a
status: milestone
output: audit done — 2 bugs found (clock skew line 87, counter reset line 134)
next:   worker-a fixing both bugs, then adding regression tests
```

---

## 4. Worker Gets Blocked → `blocked`

While fixing Bug 2, worker-a hits a dependency barrier.

**Worker-a → Main agent:**
```
status:   blocked
summary:  fixing counter reset on IP unban (line 134)
evidence: IPBanService.onUnban() callback is not exported — cannot hook from rateLimit.ts without modifying IPBanService
risk:     high — blocked task cannot proceed
next:     need decision: (A) export the callback from IPBanService, or (B) poll for ban status
```

**Main agent → User:**
```
who:    T-007 / worker-a
status: blocked
output: Bug 1 fixed (clock skew). Bug 2 blocked: IPBanService.onUnban() not exported
next:   user decision required — export callback (A) or poll for ban status (B)?
```

**User:** "Go with option A — export the callback."

---

## 5. Blocker Resolved → Back to `in_progress`, then `done`

Main agent relays the decision. Worker-a resumes and completes the task.

**Worker-a → Main agent:**
```
status:   milestone
summary:  IPBanService updated, Bug 2 fixed, regression tests written
evidence:
  - src/services/IPBanService.ts: onUnban() now exported
  - src/middleware/rateLimit.ts: line 134 updated to call onUnban()
  - tests/rateLimit.test.ts: 4 new tests added
risk:     low — change is backward-compatible
next:     running full test suite
```

**Worker-a → Main agent:**
```
status:   done
summary:  both bugs fixed, 4 regression tests added, all 47 tests passing
evidence:
  - src/middleware/rateLimit.ts (fixes at lines 87 and 134)
  - src/services/IPBanService.ts (onUnban exported)
  - tests/rateLimit.test.ts (4 new: clock-skew, unban-reset, concurrent, zero-limit)
  - test run output: 47 passed, 0 failed
risk:     none
next:     none
```

---

## 6. Main Agent Verifies → `done`

Main agent enters `reviewing` state and checks:
- [x] rateLimit.ts line 87 uses `process.hrtime()` — correct
- [x] rateLimit.ts line 134 calls `onUnban()` to reset counter — correct
- [x] IPBanService export is backward-compatible — correct
- [x] 4 new tests present, all passing — confirmed

Verification passed. Main agent reports completion.

**Main agent → User:**
```
who:    T-007 / worker-a
status: done
output:
  - 2 bugs fixed in src/middleware/rateLimit.ts
  - src/services/IPBanService.ts updated (onUnban exported)
  - 4 regression tests added (47 total, all passing)
next:   task complete
```

---

## State Transition Summary

```
planned
  → dispatching      main dispatches to worker-a
  → in_progress      worker-a: accepted (evidence: files read)
  → milestone        worker-a: audit done, 2 bugs found
  → blocked          worker-a: IPBanService.onUnban() not exported
  → in_progress      user unblocks, worker-a resumes
  → milestone        bugs fixed, tests written
  → reviewing        worker-a: done (47 tests passing)
  → done             main verifies artifacts and reports to user
```

Every state had evidence. No silent gaps. No mystery.
