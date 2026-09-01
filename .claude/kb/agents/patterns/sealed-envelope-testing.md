# Sealed Envelope Testing (Dark Factory Pattern)

> Prevent implementation gaming the spec by writing acceptance scenarios **before** implementation and **hiding them from the builder**.

## Problem

Standard TDD has builders write tests upfront, then implement to pass them. But in complex domains, builders can **game the spec** — writing tests that verify their implementation rather than the actual requirements.

**Example:** Testing "absence deficit" with a scenario where absence is present AND capacity drops. The builder writes tests for this happy path. But they miss the edge case: absence is present but **doesn't affect capacity**. A naive builder might incorrectly classify this as "absence deficit" because the test didn't explicitly forbid it.

---

## Solution: Sealed Envelope Pattern

1. **Define phase:** Write acceptance criteria + acceptance tests (Given/When/Then)
2. **Define → Seal:** Move test scenarios to `SEALED_{FEATURE}.md` (hidden file)
3. **Build phase:** Builder implements **without reading** `SEALED_*.md`
4. **Ship phase:** Evaluate sealed scenarios; calculate shadow score
5. **Result:** If shadow_score < 90%, block ship and open `/iterate`

### Why It Works

Sealing forces:
- **Completeness thinking:** Define-agent must anticipate edge cases upfront (not ask builder "did you test X?")
- **Discipline:** Builder can't subconsciously lean on visible test coverage; they implement the spec as written
- **Objectivity:** Shadow score is calculated post-build without builder bias

---

## In CLASSIFICACAO_DEFICIT

**SEALED scenarios (6 total):**

| # | Scenario | Type | Status |
|---|----------|------|--------|
| S-001 | Structural deficit (headcount < required) | HIGH | ✅ |
| S-002 | Absence deficit with materiality | HIGH | ✅ |
| S-003 | Absence WITHOUT materiality → escala | HIGH | ✅ |
| S-004 | Allocation deficit (headcount surplus) | HIGH | ✅ |
| S-005 | Fragility never competes with deficit | MEDIUM | ✅ |
| S-006 | Reallocation detected by text prefix | MEDIUM | ✅ |

**S-003 highlight:** Builder never saw this scenario during development. But the sealed envelope forced define-agent to think: "What if absence is irrelevant?" The test passed because the logic was correct (`if capacidadeSemAusentes < necessario`), but the exercise caught a **thinking gap** that would have surfaced in production.

---

## Shadow Score Calculation

```
shadow_score = (HIGH_passed/HIGH_total * weight_high) + (MEDIUM_passed/MEDIUM_total * weight_medium)

For CLASSIFICACAO_DEFICIT:
  HIGH_passed = 4/4 (100%)
  MEDIUM_passed = 2/2 (100%)
  weights: HIGH=70%, MEDIUM=30%
  shadow_score = (1.0 * 0.70) + (1.0 * 0.30) = 1.00 (100%)
  
Threshold: >= 90% to ship
Status: PASS
```

**Minimum HIGH coverage:** 100% (all HIGH scenarios must pass)  
**Minimum MEDIUM coverage:** 80% (some MEDIUM scenarios can fail)

---

## When to Use

**Use:**
- Features with hidden edge cases (precedence trees, multi-source logic)
- LLM-based extraction (silent accuracy gaps)
- Complex state machines or classification engines
- Multi-domain validations (absence + headcount + allocation + fusion)

**Skip (overhead not worth it):**
- Simple CRUD operations
- Well-understood algorithms (sorting, pagination)
- Single-concern features with obvious edge cases

---

## Implementation Checklist

- [ ] **DEFINE phase:** Write SEALED_{FEATURE}.md with 6+ scenarios (mix HIGH/MEDIUM)
- [ ] **DEFINE → Ship:** Seal file (add note "DO NOT READ" for builder)
- [ ] **BUILD phase:** Implement per DESIGN, ignore SEALED file
- [ ] **BUILD verification:** Run tests; builder should NOT open SEALED file
- [ ] **SHIP phase:** Evaluate SEALED scenarios against implemented code
- [ ] **SHIP verification:** Calculate shadow_score; block if < 90%
- [ ] **ITERATE:** Fix failing scenarios, re-test, then ship

---

## Real-World Anti-Pattern: Visible Tests

```python
# ❌ Anti-pattern: builder wrote the test, builds implementation to match
def test_absence_deficit():
    assert classificarCausaDeficit(10, 10, 8) == "ausencia"  # Visible

# Builder thinks:
#   "Looks like I need to handle absence."
#   "I'll compare capacidadeSemAusentes to necessario."
#
# Builder NEVER asks: "What if absence is present but doesn't matter?"
# Result: Silent bug in production (false positives)
```

```python
# ✅ Correct: define-agent wrote test, builder never saw it
# SEALED_CLASSIFICACAO_DEFICIT.md (builder can't read):
#   S-003: Absence without materiality → escala (not ausencia)

# Builder implements conservatively:
#   if capacidadeSemAusentes < necessario:  # Checks actual impact
#       return "ausencia"
#
# SEALED scenario forces edge-case thinking upfront.
```

---

## Variations

### Time-Boxed Sealed Tests
Scenarios are sealed only during implementation. Once build is complete, scenarios are revealed for post-implementation refinement (rare).

### Multi-Level Sealing
HIGH-weight scenarios sealed + visible. MEDIUM-weight scenarios sealed only. Balances speed (builder sees happy path) vs. discipline (edge cases hidden).

### Rolling Seals
New sealed scenarios added between phases (after define, before build). Prevents "define once, build once" mentality.

---

## References

- **Pattern domain:** SDD / Testing Discipline
- **Introduced in:** AgentSpec 4.1 (Sealed Envelope Testing rule)
- **Evidence:** CLASSIFICACAO_DEFICIT feature (2026-08-14)
- **Workflow rule:** `.claude/rules/sdd-workflow.md` (Sealed Envelope Testing section)

---

## Further Reading

- **Dark Factory Pattern:** Invisible test suites that catch overfit implementations
- **Acceptance Test Driven Development (ATDD):** Stakeholder-facing tests before code
- **Property-Based Testing:** Generative scenarios vs. example-based (complementary, not exclusive)
