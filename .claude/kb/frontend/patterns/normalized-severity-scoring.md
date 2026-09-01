# Pattern: Normalized Severity Scoring

> Normalize operational metrics by their context before applying severity thresholds.

---

## Problem

Absolute thresholds fail when the same metric means different things across different contexts:

```typescript
// BROKEN: Deficits of 8 in different sectors
if (deficitMedio > 10) severity = "critical";

// Setor A: deficitMedio=8, necessario=100 → 8% (not critical, should be mild)
// Setor B: deficitMedio=8, necessario=15 → 53% (critical!)
// But both get the same result because we ignore context
```

---

## Solution

**Step 1:** Calculate the metric relative to its context
```typescript
const relativo = deficitMedio / necessario;  // or other context variable
```

**Step 2:** Apply thresholds to the relative value, not the absolute value
```typescript
if (relativo >= 0.40) severity = "critical";
if (relativo >= 0.15) severity = "moderate";
return "mild";
```

---

## Full Example (from REDESIGN_RISCOS_ESTRUTURAIS)

```typescript
function classificarSeveridadeDeficit(
  deficitMedio: number,
  necessario: number,  // context: demand
  ocorrencias: number,
  semanasAnalisadas: number,
): Severidade {
  if (necessario <= 0) return "leve";  // guard
  
  // Normalize by context
  const relativo = deficitMedio / necessario;
  const todasAsSemanas = ocorrencias === semanasAnalisadas;
  
  // Apply thresholds to relative value
  if (relativo >= 0.4 || (relativo >= 0.25 && todasAsSemanas)) return "critico";
  if (relativo >= 0.15) return "moderado";
  return "leve";
}
```

---

## Context Variables (Common Patterns)

| Scenario | Metric | Context | Formula |
|----------|--------|---------|---------|
| Staffing shortage | deficit headcount | total demand | deficit / necessario |
| Overtime burden | hours extra | team size | hours / headcount |
| Error rate | errors | transactions | errors / volume |
| Response time | latency (ms) | SLA target | latency / target_ms |

---

## When to Use

- Any risk or alert scoring
- Multi-tenant or multi-site systems (contexts differ)
- Dashboards comparing metrics across heterogeneous units
- Any metric where "bigger is worse" but absolute values are meaningless

---

## When NOT to Use

- Single fixed context (e.g., single-tenant, single-size deployment)
- Thresholds defined by specification or standard (e.g., "max 50 errors per minute" — already normalized)

---

## Reusable Threshold Hierarchy (Severidade)

For most operational metrics:

```
CRITICAL:   relative >= 40%  OR (relative >= 25% AND condition_sustained)
MODERATE:   15% <= relative < 40%
MILD:       relative < 15%
```

Adjust the percentages per domain, but keep the structure.

---

## Testing Pattern

```typescript
describe("normalized severity", () => {
  it("same absolute value, different contexts, yields different severities", () => {
    // deficitMedio=8, necessario varies
    expect(score(8, 100)).toBe("mild");        // 8%
    expect(score(8, 15)).toBe("critical");     // 53%
  });
});
```

---

## Related

- Pattern: [Piecewise Highest Severity](./piecewise-highest-severity.md) — when combining multiple severity scores
- KB: [Workforce Management Risk Scoring](../../workforce/risk-scoring.md) — domain-specific implementation
