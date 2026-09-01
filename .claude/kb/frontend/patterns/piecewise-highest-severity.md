# Pattern: Piecewise Highest Severity

> When an entity has multiple independent risk signals, display the highest/worst severity, never an average.

---

## Problem

Multi-signal risk assessment needs to communicate: "This is bad because of X and also because of Y."

```typescript
// BROKEN: Average severity loses information
const avg = (severidadeDeficit + severidadeHe) / 2;
// If deficit="moderate" (1) and HE="critical" (0), avg=0.5 (???)
// User doesn't know if we're saying "critical" or "moderate"

// BROKEN: First-encountered severity is arbitrary
const displayed = signals[0].severity;  // depends on order
```

---

## Solution

**Always display the worst severity.**

```typescript
// Rank severities from worst to best
const ORDEM_SEVERIDADE = { critico: 0, moderado: 1, leve: 2 };

// Combine by picking the worst (lowest rank)
function piorSeveridade(a: Severidade, b: Severidade): Severidade {
  return ORDEM_SEVERIDADE[a] <= ORDEM_SEVERIDADE[b] ? a : b;
}

// Usage
const severidade = piorSeveridade(severidadeDeficit, severidadeHe);
```

---

## Full Example (from REDESIGN_RISCOS_ESTRUTURAIS)

```typescript
// In LinhaPainelRisco calculation:
const severidadeDeficit = descoberto.ocorreu 
  ? classificarSeveridadeDeficit(...)
  : null;

const severidadeHe = he_sistematica.ocorreu
  ? classificarSeveridadeHe(...)
  : null;

// Combine: if both are present, show the worst; if only one, show that one
let severidade: Severidade | null = null;
if (severidadeDeficit && severidadeHe) {
  severidade = piorSeveridade(severidadeDeficit, severidadeHe);
} else if (severidadeDeficit) {
  severidade = severidadeDeficit;
} else if (severidadeHe) {
  severidade = severidadeHe;
}
// else null (no signals with formulas)
```

---

## Principle

**A system is only as healthy as its worst component.**

If the system has:
- Moderate staffing issues (deficit)
- Critical overtime (HE)

Then the system is **critical**, not moderate.

This principle applies across domains:
- Health: patient has broken arm (moderate) + sepsis (critical) → critical
- Infra: server has high latency (moderate) + 50% error rate (critical) → critical
- Finance: low margin (moderate) + high leverage (critical) → critical

---

## Testing Pattern

```typescript
describe("piorSeveridade", () => {
  it("returns critical when either input is critical", () => {
    expect(piorSeveridade("critico", "leve")).toBe("critico");
    expect(piorSeveridade("moderado", "critico")).toBe("critico");
  });
  
  it("returns worst of moderado and leve", () => {
    expect(piorSeveridade("moderado", "leve")).toBe("moderado");
  });
  
  it("is idempotent when inputs are equal", () => {
    expect(piorSeveridade("moderado", "moderado")).toBe("moderado");
  });
  
  it("is commutative (order doesn't matter)", () => {
    expect(piorSeveridade("critico", "leve")).toBe(piorSeveridade("leve", "critico"));
  });
});
```

---

## Variants

### Three or More Signals

```typescript
const severidades = [
  descoberto.ocorreu ? classificarSeveridadeDeficit(...) : null,
  he.ocorreu ? classificarSeveridadeHe(...) : null,
  fusao.ocorreu ? "leve" : null,  // no formula for fusion
];

const severidade = severidades
  .filter(s => s !== null)
  .reduce((worst, current) => piorSeveridade(worst, current));
```

### Weighted Severity (Rare)

If signals have different importance (usually they don't), use weights:

```typescript
const weighted = [
  { severity: severidadeDeficit, weight: 0.6 },
  { severity: severidadeHe, weight: 0.4 },
];

// Still take the worst, but consider weights in display color intensity
// e.g., critical + critical = darker red than critical + moderate
```

---

## Common Mistakes

1. **Averaging:** `(0 + 2) / 2 = 1` (ambiguous)
2. **String concatenation:** `"critico+moderado"` (confusing)
3. **First-encountered:** Depends on order (non-deterministic)
4. **Ignoring nulls:** If no signal is active, severity is `null`, not `leve`

---

## Related

- Pattern: [Normalized Severity Scoring](./normalized-severity-scoring.md) — each individual signal's severity
- KB: [Risk Dashboard Design](../../frontend/risk-dashboard-design.md)
