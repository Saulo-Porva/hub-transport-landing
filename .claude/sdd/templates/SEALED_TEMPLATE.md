# SEALED: {Feature Name}

> Acceptance scenarios sealed from the build phase.
> **BUILDER AGENT: DO NOT READ THIS FILE BEFORE IMPLEMENTATION IS COMPLETE.**
> Written by define-agent as the last step of Phase 1; evaluated by ship-agent after build.

## Metadata

| Attribute | Value |
|-----------|-------|
| **Feature** | {FEATURE_NAME} |
| **Created** | {YYYY-MM-DD} |
| **Author** | define-agent |
| **Evaluated** | No |
| **Shadow Score** | Pending |

---

## Purpose

These scenarios are the "holdout set" — the builder agent never reads them during implementation.
If the builder passes without having seen them, it means the implementation truly satisfies requirements
rather than being tuned to visible tests.

**Rule:** `build-agent` must never read `SEALED_*.md`. `ship-agent` evaluates them post-build.

---

## Sealed Scenarios

### Scenario S-001: {Happy Path — Descriptive Name}

**Setup (exact system state):**
```text
{Complete preconditions: data that must exist in DB, sessions, env variables, mocks active}
```

**Steps:**
1. {Exact action 1}
2. {Exact action 2}
3. {Exact action 3}

**Expected Outcomes:**
- [ ] {Observable result — what user/caller sees}
- [ ] {Persistence — what is written to DB / Storage}
- [ ] {Side effects — events emitted, emails sent, etc.}

**Weight:** HIGH — must pass to ship

---

### Scenario S-002: {Error / Rejection Case — Descriptive Name}

**Setup:**
```text
{State that triggers the error path}
```

**Steps:**
1. {Action that should fail}

**Expected Outcomes:**
- [ ] {Error message or status code returned}
- [ ] {System state unchanged (no partial writes)}
- [ ] {Audit log or error logged correctly}

**Weight:** HIGH — must pass to ship

---

### Scenario S-003: {Edge Case or Boundary — Descriptive Name}

**Setup:**
```text
{Extreme or unusual state}
```

**Steps:**
1. {Edge action}

**Expected Outcomes:**
- [ ] {Correct handling of boundary}
- [ ] {No crash or silent failure}

**Weight:** MEDIUM

---

### Scenario S-004: {Rollback / Consistency Case — Descriptive Name}

**Setup:**
```text
{State mid-operation or partial failure injected}
```

**Steps:**
1. {Trigger partial failure}

**Expected Outcomes:**
- [ ] {System returns to consistent state}
- [ ] {No orphaned data in any store}

**Weight:** MEDIUM (HIGH if feature writes to multiple systems)

---

> Add `### Scenario S-00N:` for each additional acceptance test beyond the minimum 4.
> Every AT from DEFINE must have a corresponding sealed scenario here.

---

## Shadow Score Calculation

| Weight | Scenarios | Threshold |
|--------|-----------|-----------|
| HIGH | S-001, S-002 | All must pass (bloqueante) |
| MEDIUM | S-003, S-004 | ≥ 80% must pass |
| LOW | S-005+ | Best effort |

```
shadow_score = (HIGH_passed/HIGH_total * 0.60) + (MEDIUM_passed/MEDIUM_total * 0.30) + (LOW_passed/LOW_total * 0.10)
```

**Minimum to ship:** 90% (weighted)

---

## Evaluation Instructions (for ship-agent)

1. Read each scenario above — do not share with the team until after evaluation
2. Test each step against the implemented code (manually or via automated runner)
3. Mark each outcome checkbox ✅ Pass or ❌ Fail
4. Calculate shadow_score using the formula above
5. Record results in `SHIPPED_{FEATURE}.md` → Shadow Score section
6. If shadow_score < 90%: flag blockers, do not ship — open `/iterate` to address gaps
