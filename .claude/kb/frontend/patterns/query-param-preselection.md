# Query Param Pre-Selection Pattern

## Overview

Navigate to a form with a pre-selected field via URL query param, using `searchParams` (Next.js 15 Server Components) + `defaultValue` in form inputs.

**Usage:** When a preceding page/feature knows which entity the user wants to work with and wants to skip manual field selection.

**Benefit:** Reduces form friction (eliminates manual search through large lists), improves workflow efficiency.

---

## Problem

Multi-step workflows often require users to:
1. Click "Action" on an item (e.g., "Configurar demanda" on setor)
2. Navigate to a form on a different page
3. Manually search/select the same entity from a large dropdown (10-50+ options)

This adds 5-10 clicks per workflow and risks selecting the wrong entity.

---

## Solution

Encode the entity ID in the URL query param, then pre-select it in the target form.

```
Page A (item detail) → Page B (form) with ?entity_id=<value>
                       Page B reads param
                       Page B pre-selects in form
```

---

## Implementation

### 1. Source Component (Item Detail Page)

Generate a link with the entity ID as query param:

```tsx
// Example: DetalheDiagnosticoSetor.tsx
export function DetalheDiagnosticoSetor({ setor }: Props) {
  return (
    <div>
      <AcaoLink href={`/sazonalidade?setor=${setor.setor_id}`}>
        Configurar demanda
      </AcaoLink>
    </div>
  );
}
```

**Key Points:**
- Entity ID comes from server-side props (not user input)
- Link is generated server-side (safe from injection)
- No validation needed here (validation happens in target form)

---

### 2. Target Page (Server Component)

Extract query param from `searchParams` (Next.js 15 Promise):

```tsx
// Example: sazonalidade/page.tsx
interface PageProps {
  searchParams: Promise<{ setor?: string }>;
}

export default async function SazonalidadePage({ searchParams }: PageProps) {
  // Extract entity ID from query param
  const { setor: setorIdPreSelecionado } = await searchParams;

  // Pass to form component
  return (
    <NovaBaselineForm
      setores={setores}
      action={salvarBaselineDemanda}
      setorIdPreSelecionado={setorIdPreSelecionado}
    />
  );
}
```

**Key Points:**
- `searchParams` is a Promise (Next.js 15 Server Component)
- Always `await searchParams` before destructuring
- Value is a string (query params are always strings)
- Can be `undefined` if param is not present

---

### 3. Form Component (Client Component)

Accept optional entity ID prop, apply as `defaultValue`:

```tsx
// Example: NovaBaselineForm.tsx
"use client";

interface NovaBaselineFormProps {
  setores: Array<{ id: string; nome: string }>;
  action: (prevState: ActionResult, formData: FormData) => Promise<ActionResult>;
  setorIdPreSelecionado?: string; // ← from query param
}

export function NovaBaselineForm({
  setores,
  action,
  setorIdPreSelecionado,
}: NovaBaselineFormProps) {
  return (
    <form action={action}>
      <Select
        name="setor_id"
        required
        defaultValue={setorIdPreSelecionado}
      >
        {setores.map((s) => (
          <option key={s.id} value={s.id}>
            {s.nome}
          </option>
        ))}
      </Select>
    </form>
  );
}
```

**Key Points:**
- Prop is optional (`setorIdPreSelecionado?`)
- Apply directly to form element: `defaultValue={setorIdPreSelecionado}`
- No validation needed (browser validates)

---

## Validation & Fallback

### For `<Select>` (HTML native)

If `defaultValue` does not match any `<option value>`:
- Browser silently ignores it (spec-compliant behavior)
- First `<option>` is selected automatically
- No error thrown

**When to use:** Pre-selecting from a hardened list of options (server-rendered)

```tsx
// Safe — options come from server, no injection risk
<select defaultValue={maybeInvalidId}>
  {serverSetores.map(s => <option value={s.id}>{s.nome}</option>)}
</select>
```

### For `<Input type="text">`

If `defaultValue` is invalid, user sees the value and can clear/edit:
- No validation happens automatically
- Consider if you need client-side validation

```tsx
// User can see and edit
<input type="text" defaultValue={maybeInvalidId} />
```

---

## URL Param Naming

Use consistent param names across your app:

| Param | Used In | Example |
|-------|---------|---------|
| `?setor=` | All setor-related forms | `/sazonalidade?setor=123` |
| `?funcionario=` | Employee forms | `/turnos?funcionario=abc` |
| `?unidade=` | Navigation | `/escalas?unidade=xyz` |

**Convention:** Use singular entity name (`setor`, not `setor_id`).

---

## Security Considerations

### Safe Usage

- Entity IDs from server-side props (links generated server-side)
- Hardened options list (server-rendered select options)
- Browser validation for selects (HTML native)

✅ **No injection risk** because:
1. Entity ID is not user input (comes from your database)
2. Form options are not derived from the query param (hardened list)
3. Browser validates select values against available options

### Unsafe Usage (DON'T DO)

```tsx
// ❌ Wrong: Using query param to populate options
const options = JSON.parse(decodeURIComponent(searchParams.options));

// ❌ Wrong: No validation on free-text field
<input defaultValue={searchParams.userInput} />

// ❌ Wrong: Storing user-provided ID without validation
const filteredResults = data.filter(d => d.id === searchParams.entityId);
```

---

## Common Patterns in This Codebase

| Page | Source | Target | Param | Status |
|------|--------|--------|-------|--------|
| `/diagnostico` → `/sazonalidade` | DetalheDiagnosticoSetor | NovaBaselineForm | `?setor=` | ✅ Active (SETOR_SEM_DEMANDA_CTA feature) |
| `/funcionarios` (menu) → `/turnos` | Navigation | TurnoForm | `?funcionario=` | ✅ Likely existing pattern |

---

## Testing

### Test Case 1: Param Present, Valid

```gherkin
Given the param ?setor=valid-uuid
When the form loads
Then the corresponding setor is selected in the dropdown
```

### Test Case 2: Param Absent

```gherkin
Given no ?setor= param in URL
When the form loads
Then the first option is selected (default behavior)
And the page behaves identically to before the param was added
```

### Test Case 3: Param Present, Invalid

```gherkin
Given the param ?setor=invalid-uuid-not-in-list
When the form loads
Then no error is thrown
And the first option is selected (browser fallback)
```

---

## Performance Notes

- `searchParams` is a Promise in Next.js 15 — use `await`
- No additional DB query needed (options list is already fetched)
- Browser validation is instant (no server round-trip)

---

## Examples in This Project

**Feature:** SETOR_SEM_DEMANDA_CTA (2026-08-16)

**Files:**
- Source: `src/app/(app)/diagnostico/DetalheDiagnosticoSetor.tsx` (line 118)
- Page: `src/app/(app)/sazonalidade/page.tsx` (lines 43-45)
- Form: `src/app/(app)/sazonalidade/NovaBaselineForm.tsx` (lines 18, 41)

**Usage:**
- Click "Configurar demanda" on a setor in Diagnóstico
- Navigates to `/sazonalidade?setor=<id>`
- Setor is pre-selected in the baseline form

---

## Limitations

1. **Query param length:** URLs have length limits (~2000 chars). For IDs only, not a concern. If encoding objects, use POST instead.
2. **Refresh behavior:** If user bookmarks the URL, the pre-selection persists (may be good or bad depending on context).
3. **Multiple selects:** Pattern is designed for 1-2 params. For more complex pre-selection, consider using localStorage or a more structured state pattern.

---

## See Also

- [Next.js 15 searchParams Documentation](https://nextjs.org/docs/app/api-reference/functions/use-search-params)
- [HTML Select Element Spec](https://html.spec.whatwg.org/#the-select-element)
- `.claude/kb/nextjs/` — Next.js patterns in this project

---

**Status:** Active pattern (as of 2026-08-16)  
**Feature Origin:** SETOR_SEM_DEMANDA_CTA
