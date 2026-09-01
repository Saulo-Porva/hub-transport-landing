# Pattern: Severity Band Reuse Across Metrics

> When classifying severity on a new axis, reuse validated thresholds instead of inventing new ones.

---

## Problem

A product has validated severity thresholds for one metric (e.g., coverage deficit at 40%/15%). A new feature introduces a related but distinct metric (e.g., coverage fragility). There's a temptation to tune the new thresholds independently, "because this metric is different." This leads to:

- Metric drift (different severity definitions for conceptually similar problems)
- Inconsistent user expectations (Why is deficit at 35% "critical" but fragility at 35% "moderate"?)
- Wasted specialist time (re-validating thresholds that were already validated)

---

## Solution

Reuse the validated thresholds as constants. If domain experts confirm the new metric is on the same axis (relative gap, not absolute), apply the same classification formula.

### Example: Reusing Deficit Thresholds for Coverage Fragility

**Validated for deficit (2025, specialist review):**
```ts
export function classificarSeveridadeDeficit(
  deficitMedio: number,
  necessario: number,
  ...
): Severidade {
  const relativo = deficitMedio / necessario;
  if (relativo >= 0.40) return "critico";      // 40%+ gap
  if (relativo >= 0.15) return "moderado";     // 15%+ gap
  return "leve";
}
```

**Thresholds:** 0.40 (critical), 0.15 (moderate) — already field-tested

**New metric, same axis (coverage fragility):**
```ts
export function classificarSeveridadeCoberturaFragil(
  coberturaSemRemanejamentoPct: number,
): Severidade {
  const relativo = 1 - coberturaSemRemanejamentoPct;
  // Same as deficit, just inverted (1 - coverage = gap)
  if (relativo >= 0.40) return "critico";      // 40%+ of coverage depends on remanejamento
  if (relativo >= 0.15) return "moderado";
  return "leve";
}
```

**Benefits:**
- User sees consistent severity definitions
- No new specialist review needed (already validated for this axis)
- Code review is trivial: "reuses existing thresholds, formula is inverted coverage"

---

## When to Reuse Thresholds

✅ **Reuse if:**
- The new metric is on the same axis (both are "gap %" or "utilization %")
- Domain experts confirm the semantic equivalence ("fragility is a type of gap")
- The existing threshold has been in production for >3 months
- You can articulate the conceptual link between the metrics

❌ **Don't reuse if:**
- The new metric is on a fundamentally different axis (e.g., defect rate vs. time-to-resolution)
- Specialist validation suggests different severity profiles
- The existing threshold is known to be provisional ("temporary, pending review")
- You're just guessing because thresholds are numbers

---

## How to Validate Reuse

Before shipping a feature that reuses thresholds, get specialist sign-off:

1. **Email/Slack specialist:** "New metric X is conceptually a [gap/utilization/ratio]. Proposing to classify it using existing thresholds 0.40/0.15. Does this make sense?"
2. **Document the decision:** In DESIGN doc, add a note: "Severity bands reuse existing thresholds (validated 2025-XX-YY) because both are gap percentages."
3. **Test edge cases:** Ensure that a metric at exactly 0.40 returns "critico" (not "moderado"), matching existing behavior.

---

## Anti-Pattern: Inventing New Thresholds

❌ **Don't do this:**
```ts
// Initial deficit classification
export function classificarSeveridadeDeficit(...): Severidade {
  if (relativo >= 0.40) return "critico";  // validated 2025
  if (relativo >= 0.15) return "moderado";
  return "leve";
}

// Later: new metric, new thresholds (uh-oh)
export function classificarSeveridadeCoberturaFragil(pct: number): Severidade {
  if (pct < 0.35) return "critico";   // <-- Why 0.35? Never explained!
  if (pct < 0.20) return "moderado";  // <-- Different from 0.15!
  return "leve";
}
```

**Problems:**
- Three different severity definitions (0.40 deficit, 0.35 coverage, 0.20 HE) float around the codebase
- Code reviewers don't know if this was intentional or a typo
- Next engineer adds a 4th metric and invents 0.33 "just to be safe"
- Specialist eventually notices the drift and we rework everything

---

## Implementation Checklist

```
[ ] Identify validated threshold source (existing function)
[ ] Confirm with specialist: "Same axis?"
[ ] Create new function as separate (don't modify existing)
[ ] Use the same numeric constants (0.40, 0.15) — no copy-paste with tweaks
[ ] Add comment linking to the original:
    "Reuses DEFICITARIO_THRESHOLD_CRITICAL (0.40) — same axis (relative gap %)"
[ ] Test that new metric at 0.40 returns "critico" (matches original behavior)
[ ] Note in feature DESIGN doc: "Severity bands reused, validated [date]"
[ ] Ship with confidence
```

---

## Bonus: Creating the First Threshold

If you're defining a severity classification for the *first* time, involve a specialist:

1. **Gather data:** Real production examples of the metric at different severity levels
2. **Interview specialists:** "When do you get worried about this metric?"
3. **Set thresholds based on that conversation, not arbitrary numbers**
4. **Document the decision:** "0.40 chosen because [business reason], validated [date]"
5. **Plan review:** "We'll revisit this in 6 months based on false-positive rate"

---

## References

- **Codebase:** `src/lib/escala/severidade-risco.ts`
  - `classificarSeveridadeDeficit` (original, validated)
  - `classificarSeveridadeCoberturaFragil` (reuses thresholds, added 2026-08-14)
- **Feature:** RISCOS_ESTRUTURAIS_EVIDENCIAS (2026-08-14)
- **Domain validation:** Communication with otimizacao-matematica-specialist
- **Related patterns:** derived-annotation-pattern.md, real-vs-theoretical-data.md

---

## Next Steps

When defining a new severity metric:

1. Search codebase for existing severity functions
2. Assess whether the new metric is on the same axis
3. If yes, file the specialist confirmation (Slack message, email) and document it
4. Reuse the thresholds, no new numbers
5. If unsure, ask before coding — don't guess
