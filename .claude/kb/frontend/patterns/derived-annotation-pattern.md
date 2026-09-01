# Pattern: Derived Annotations Without Core Logic Modification

> When enriching a metric with additional context, keep changes additive to avoid regression.

---

## Problem

A feature detects a pattern at coarse granularity (e.g., daily deficit). A later enhancement requests finer context (e.g., hourly window of that deficit). The temptation is to modify the core detection function to work at the new granularity. This risks breaking existing consumers and introduces unnecessary coupling.

---

## Solution

Create a separate **derived** function that operates on top of the core result, without modifying the original detection logic.

### Example: Hourly Window for Daily Deficit Detection

**Core detection (unchanged):**
```ts
export function calcularDescobertoEstrutural(
  alocacoesSetor: AlocacaoParaRisco[],
  metasSetor: MetaParaRisco[],
  ...
): EstatisticaRisco {
  // Returns: ocorreu, ocorrencias, deficitMedio, deficitMaximo, serieSemanal
  // Granularity: day of week
}
```

**Derived annotation (new, additive):**
```ts
export function construirCobertosPorChaveHora(
  alocacoesSetor: AlocacaoParaRisco[],
  janelaInicioISO: string,
  janelaSemanas: number,
): Map<string, Set<string>> {
  // Granularity: day + hour
  // Used only to annotate, not to redefine detection
}

export function calcularFaixaHorariaAfetada(
  diaSemanaAlvo: number,
  semanasComGap: number[],           // derived from existing serieSemanal
  metasSetorPorHora: MetaParaRisco[],
  cobertosPorChaveHora: Map<string, Set<string>>,
  janelaSemanas: number,
): FaixaHoraria | null {
  // Returns hourly window only for the day + weeks already identified as having gaps
  // Zero impact on detection itself
}
```

**Integration:**
```ts
export function calcularDescobertoEstrutural(...): EstatisticaRisco {
  // ... existing logic, unchanged ...

  // Derive annotation *after* detection is complete
  if (diaSemanaMaisFrequente !== null) {
    const semanasComGap = serieSemanal
      .map((deficit, semana) => deficit > 0 ? semana : -1)
      .filter(s => s >= 0);
    const cobertosPorChaveHora = construirCobertosPorChaveHora(...);
    const faixaHorariaAfetada = calcularFaixaHorariaAfetada(
      diaSemanaMaisFrequente,
      semanasComGap,
      metasSetor,
      cobertosPorChaveHora,
      janelaSemanas,
    );
  }

  return { ...existing fields..., faixaHorariaAfetada };
}
```

---

## Benefits

| Benefit | Why |
|---------|-----|
| **Zero regression risk** | Core consumers never see the change; old tests continue to pass |
| **Optional enrichment** | Annotation can be null (defensive); callers check before using |
| **Single responsibility** | Detection ≠ annotation; each function does one thing |
| **Easier code review** | Reviewer sees "new function, new field, used here" — no surprises in existing logic |
| **Composable** | Future features can reuse the annotation function for different core patterns |

---

## When to Use

✅ **Use this pattern when:**
- The core metric is already working and tested
- You want to add context (finer granularity, derived field, visual annotation)
- Multiple consumers depend on the core function
- The annotation is optional (can be null)

❌ **Don't use this pattern when:**
- The core detection itself is wrong and needs redefinition (use `/iterate`)
- The annotation is mandatory for the core calculation
- The feature is small enough to refactor the core safely (low consumer count)

---

## Variants

### Variant A: Computed Field in the Result Type

If the annotation is simple enough, add it as a computed field:

```ts
export interface EstatisticaRisco {
  ocorrencias: number;
  // ... existing fields ...
  faixaHorariaAfetada: FaixaHoraria | null;  // NEW, optional, populated only if ocorreu
}
```

### Variant B: Separate Query/Fetch

For heavier annotations, consider a separate query layer:

```ts
// Core detection: from real data (already cached)
const deteccao = calcularDescobertoEstrutural(...);

// Annotation: lazy-fetched only when needed (e.g., drawer opens)
const anotacao = faixa 
  ? await buscarFaixaHorariaDetalhada(deteccao.dia, deteccao.semanas)
  : null;
```

---

## References

- **Codebase:** `src/lib/escala/painel-risco.ts` — functions `construirCobertosPorChaveHora`, `calcularFaixaHorariaAfetada`
- **Feature:** RISCOS_ESTRUTURAIS_EVIDENCIAS (2026-08-14)
- **Related patterns:** severity-band-reuse.md

---

## Next Steps

When applying this pattern to a new feature:

1. Identify the core detection function
2. List existing consumers (search for function name in codebase)
3. Design the derived function to operate on the core result, not redefine it
4. Add the new field to the result type as `| null` (optional)
5. Test that core consumers' tests still pass unchanged
