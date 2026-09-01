# Pattern: Client-Side Type Filtering on Pre-Loaded Data

> For datasets under ~200 items, use `useState` to filter by type client-side instead of query params + server call.

---

## Problem

Naive approach: Each type filter triggers a server query.

```typescript
// EXPENSIVE: 4 separate server calls
const handleFilterChange = async (type) => {
  const data = await fetch(`/api/risks?type=${type}`);
  setRisks(data);
};

// Downsides:
// - Network latency (100-500ms per filter change)
// - Loading state complexity
// - Server load for trivial filtering
// - No responsive UX ("filter in progress...")
```

---

## Solution

**Load all data once, filter in-memory with `useState`.**

```typescript
const [filterType, setFilterType] = useState<FiltroTipo>("todos");

const filtered = linhas.filter(linha => passaNoFiltro(linha, filterType));

function passaNoFiltro(linha: LinhaPainelRisco, filtro: FiltroTipo): boolean {
  if (filtro === "todos") return true;
  if (filtro === "deficit") return linha.descoberto_estrutural.ocorreu;
  if (filtro === "fragil") return linha.cobertura_fragil;
  return linha.he_sistematica.ocorreu;  // "he"
}
```

**Result:**
- Instant filter response (< 10ms)
- No loading state needed
- No server calls
- Smooth UX

---

## Full Example (from REDESIGN_RISCOS_ESTRUTURAIS)

```typescript
export function RiscosEstruturais({ linhas }: RiscosEstruturaisProps) {
  const [filtroTipo, setFiltroTipo] = useState<FiltroTipo>("todos");

  // All data arrives as prop (loaded once, server-side)
  const linhasComRisco = linhas.filter(
    (l) => l.cobertura_via_fusao || l.descoberto_estrutural.ocorreu 
        || l.he_sistematica.ocorreu || l.cobertura_fragil,
  );

  // Filter and sort client-side
  const linhasExibidas = linhasComRisco
    .filter((l) => passaNoFiltro(l, filtroTipo))
    .sort((a, b) => ordemExibicao(a.severidade) - ordemExibicao(b.severidade) 
                  || a.setor_nome.localeCompare(b.setor_nome));

  return (
    // Render buttons to change filtroTipo (triggers re-render instantly)
    {OPCOES_FILTRO.map((opcao) => (
      <button onClick={() => setFiltroTipo(opcao.valor)}>
        {opcao.rotulo}
      </button>
    ))}
    // Render filtered list
    {linhasExibidas.map((linha) => (...))}
  );
}
```

---

## When to Use

**Use client-side filtering if:**
- Dataset size: < ~200 items
- Filtering logic: simple boolean predicates (no aggregation, no joins)
- User interaction: frequent filter changes (every keystroke)
- UX: need instant feedback without loading state

**Examples:**
- Type/category filters (this pattern)
- Date range pickers (pre-load last 30 days, filter in UI)
- Search on names (pre-load all, `filter(item => item.name.includes(query))`)

---

## When NOT to Use

**Use server-side filtering (query params) if:**
- Dataset size: > 1000 items (can't render all in DOM anyway)
- Filtering logic: expensive (requires aggregation, joins, complex queries)
- Data freshness: filter results change frequently (need live data)
- Network: unreliable or high-latency (user expects loading state anyway)

**Examples:**
- Full-text search on large database
- Aggregated metrics ("orders per region")
- Real-time data (stock prices, active users)

---

## Implementation Checklist

- [ ] Load all data once via prop or SSR
- [ ] Define filter type/enum (e.g., `type FiltroTipo = "todos" | "deficit" | ...`)
- [ ] Implement `passaNoFiltro()` or `filter()` predicate
- [ ] Use `useState<FiltroTipo>()` for current selection
- [ ] Call `.filter()` on the dataset before rendering
- [ ] Render filter buttons that call `setFiltroTipo()`
- [ ] Test: filter changes don't cause network calls (check Network tab)

---

## Performance Notes

JavaScript `.filter()` on 200 items: ~0.1ms (negligible)  
Rendering 200 items: ~10–50ms (depends on component complexity)

**Total filter response time: < 100ms** (feels instant)

For larger datasets:
- Consider virtualization (render only visible items)
- Add debouncing if filter is derived from text input
- Profile with React DevTools Profiler

---

## Testing

```typescript
describe("RiscosEstruturais filtering", () => {
  it("shows only HE rows when filter is 'he'", () => {
    render(<RiscosEstruturais linhas={mockRiscos} />);
    
    const heButton = screen.getByRole("button", { name: /HE sistemática/i });
    fireEvent.click(heButton);
    
    const rows = screen.getAllByRole("region");  // or your row selector
    rows.forEach(row => {
      expect(row).toHaveAttribute("data-has-he", "true");
    });
  });

  it("restores full list when filter is set back to 'todos'", () => {
    // ... filter to 'he', then back to 'todos'
    expect(screen.getAllByRole("region")).toHaveLength(mockRiscos.length);
  });

  it("updates instantly without loading state", () => {
    render(<RiscosEstruturais linhas={mockRiscos} />);
    
    const startTime = performance.now();
    fireEvent.click(screen.getByRole("button", { name: /Déficit/i }));
    const duration = performance.now() - startTime;
    
    expect(duration).toBeLessThan(50);  // instant, no loading
  });
});
```

---

## Variants

### With Sorting

```typescript
const sorted = filtered
  .sort((a, b) => a.createdAt - b.createdAt)
  .sort((a, b) => PRIORITY[a.type] - PRIORITY[b.type]);  // sort by type, then date
```

### With Multi-Select

```typescript
const [selectedTypes, setSelectedTypes] = useState<Set<FiltroTipo>>(new Set(["todos"]));

const filtered = selectedTypes.has("todos")
  ? linhas
  : linhas.filter(l => selectedTypes.some(type => passaNoFiltro(l, type)));
```

### With Search

```typescript
const [filterType, setFilterType] = useState<FiltroTipo>("todos");
const [searchQuery, setSearchQuery] = useState("");

const filtered = linhas
  .filter(l => passaNoFiltro(l, filterType))
  .filter(l => l.setor_nome.toLowerCase().includes(searchQuery.toLowerCase()));
```

---

## Related

- KB: [Next.js Client Components](../../nextjs/client-components.md) — using `useState`
- Pattern: [Debounced Search Input](../patterns/debounced-search-input.md) — for text-based filtering
