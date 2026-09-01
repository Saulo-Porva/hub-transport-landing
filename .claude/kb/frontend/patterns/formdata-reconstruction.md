# Pattern: FormData Reconstruction for Client-Side State

> Building a client-side object from submitted form data instead of querying the server again.

**Status:** Extracted from `MODAL_CADASTRO_EMPREGADO` (shipped 2026-08-14)

---

## Problem

In a create→edit workflow within the same session:
1. User fills out a form and clicks "Create"
2. Server Action validates, saves to DB, returns `{ id }`
3. You want to transition to "edit mode" and keep the modal open
4. Editable fields should show the values the user just submitted, plus new data from the server

**Naive approach:** Query the server again to fetch the full object.
```typescript
// Fetches from DB after we JUST submitted the same data
const result = await criarFuncionario({ nome: "Alice", cargo: "Manager", ... });
const full = await listarFuncionario(result.id); // N+1 query
```

**Better approach:** Reconstruct the object from the FormData that's already in memory.
```typescript
const result = await criarFuncionario(formData);
const funcionario = funcionarioFromFormData(result.id, formData);
```

---

## Solution

### Step 1: Define the Reconstruction Function

Keep it **pure** (no imports of server-only libraries):

```typescript
export interface FuncionarioBasico {
  id: string;
  nome: string;
  cargo: string;
  setor_id: string;
  jornada_semanal_horas: number;
  // ... all editable fields
}

/**
 * Reconstruct a FuncionarioBasico from FormData submitted to the server.
 * Called after Server Action success to populate edit-mode UI without re-querying.
 */
function funcionarioFromFormData(id: string, formData: FormData): FuncionarioBasico {
  const nomeSocial = formData.get("nome_social");
  const matricula = formData.get("matricula");
  
  return {
    id,
    nome: String(formData.get("nome") ?? ""),
    cargo: String(formData.get("cargo") ?? ""),
    setor_id: String(formData.get("setor_id") ?? ""),
    jornada_semanal_horas: Number(formData.get("jornada_semanal_horas") ?? 0),
    nome_social: nomeSocial ? String(nomeSocial) : null,
    matricula: matricula ? Number(matricula) : null,
    // ... for each field in the form
  };
}
```

### Step 2: Extract FormData After Success

In the form's success handler:

```typescript
function handleSuccess(state: ActionResult<{ id: string }>) {
  const id = funcionario?.id ?? state.data?.id;
  if (!formRef.current || !id) return;
  
  // Extract from the form ref that was just submitted
  const reconstructed = funcionarioFromFormData(id, new FormData(formRef.current));
  onSuccess(reconstructed);
}
```

### Step 3: Use the Reconstructed Object

Pass it to the parent component for state updates:

```typescript
const [funcionarioAtual, setFuncionarioAtual] = useState<FuncionarioBasico | undefined>(
  funcionario,
);

function handleSuccess(state) {
  const reconstructed = funcionarioFromFormData(...);
  setFuncionarioAtual(reconstructed); // Now available for edit-mode tabs
}
```

---

## Why This Works

1. **User submitted data is in memory** via the FormData from `formRef.current`
2. **Server only needed to validate + save + return ID**, not to re-fetch
3. **Latency cut in half** (one round-trip instead of two)
4. **Guarantees consistency** (the object matches what the server just saved)

---

## Trade-Offs

### Pro
- **Performance:** Eliminates N+1 query (create + fetch becomes just create)
- **Offline-first friendly:** Data is already available client-side
- **Predictable:** No race condition where server state differs from what user submitted

### Con
- **Type safety:** Must align form field names with object keys perfectly
- **Hidden defaults:** If server mutates fields (e.g., auto-slugify a `name` field), the reconstruction won't reflect that
- **Maintenance burden:** If form fields change, reconstruction function must change too

**Rule of thumb:** Use this pattern only when:
- The server doesn't mutate user-submitted fields
- You trust the form validation (Pydantic, etc.)
- Latency matters (e.g., poor network, high-scale system)

---

## Example: Full Flow

```typescript
// In DadosBasicosTab.tsx
export function DadosBasicosTab({ funcionario, action, onSuccess }: ...) {
  const formRef = useRef<HTMLFormElement>(null);

  function handleSuccess(state: ActionResult<{ id: string }>) {
    const id = funcionario?.id ?? state.data?.id;
    if (!formRef.current || !id) return;
    
    // Reconstruct without querying server again
    const reconstructed = funcionarioFromFormData(id, new FormData(formRef.current));
    onSuccess(reconstructed);
  }

  return (
    <ServerActionForm action={action} formRef={formRef} onSuccess={handleSuccess}>
      <Field label="Nome">
        <Input name="nome" defaultValue={funcionario?.nome ?? ""} required />
      </Field>
      <Field label="Cargo">
        <Input name="cargo" defaultValue={funcionario?.cargo ?? ""} required />
      </Field>
      {/* ... more fields */}
      <SubmitButton>{funcionario ? "Salvar" : "Cadastrar"}</SubmitButton>
    </ServerActionForm>
  );
}

// In page.tsx
<DadosBasicosTab
  action={criarFuncionario}
  onSuccess={(criado) => {
    // `criado` is reconstructed from form data
    // Now transition to edit mode with the new object
    setModoAtual("editar");
    setFuncionarioAtual(criado);
  }}
/>
```

---

## Pitfalls

### 1. Server Mutates Fields

If your Server Action auto-processes fields (e.g., slugifies, trims, transforms), the reconstruction won't reflect that:

```typescript
// Server does:
const email = input.email.toLowerCase().trim();

// Reconstruction keeps what user typed:
const reconstructed = { email: formData.get("email") }; // "  Alice@EXAMPLE.COM  "
```

**Solution:** Have the server return the full record in the `ActionResult`, not just ID.

### 2. Null/Empty Handling

Be explicit about how empty fields are handled:

```typescript
// Ambiguous:
nome_social: formData.get("nome_social") || null,

// Clear:
nome_social: formData.get("nome_social") ? String(formData.get("nome_social")) : null,
```

### 3. Type Safety

Ensure TypeScript catches field mismatches:

```typescript
interface FuncionarioBasico {
  id: string;
  nome: string;
  // ... all required fields
}

function funcionarioFromFormData(...): FuncionarioBasico {
  // TS will error if any field is missing or wrong type
  return { id, nome, /* ... */ };
}
```

---

## Alternatives

### Alt 1: Server Returns Full Record

Have the Server Action return the complete record:

```typescript
async function criarFuncionario(input: CreateInput): Promise<ActionResult<FuncionarioBasico>> {
  const record = await db.insert(...);
  return { data: record }; // Return full object, not just { id }
}
```

**Pros:** Server is source of truth; no reconstruction needed
**Cons:** Slightly larger payload; requires server to construct full response

### Alt 2: Client-Side Caching

Store submitted form data in React state/Context to avoid losing it:

```typescript
const [formCache, setFormCache] = useState<FormData | null>(null);

// On form change (if using controlled inputs), store in cache
// On success, use cached data + server ID to reconstruct
```

**Pros:** Handles mutations, no N+1 query
**Cons:** More complex state management

### Alt 3: Accept the N+1 Query

Query the server after creation if latency isn't critical:

```typescript
const result = await criarFuncionario(...);
const full = await listarFuncionario(result.id);
setFuncionarioAtual(full);
```

**Pros:** Guaranteed consistency with server
**Cons:** Slower UX; requires second round-trip

---

## When to Use

| Scenario | Use This Pattern? | Why |
|----------|---|---|
| Create → Edit in same session, fast UX needed | ✅ Yes | Zero latency improvement; user data is at hand |
| Create → back to list (no immediate edit) | ❌ No | No reason to reconstruct; just close modal |
| Server auto-processes fields (slugs, timestamps) | ⚠️ Maybe | Only if you fetch server's response and return it |
| Offline-first / low-connectivity app | ✅ Yes | Reconstruction fits offline-first mental model |
| Very simple form (1–2 fields) | ⚠️ Overkill | Type safety overhead not worth it |

---

## References

- **Implementation:** `MODAL_CADASTRO_EMPREGADO` feature (2026-08-14)
  - `src/app/(app)/funcionarios/DadosBasicosTab.tsx` (lines 36–79)
  - `src/app/(app)/funcionarios/FuncionarioModal.tsx` (lines 61–68)
- **Related pattern:** `tabbed-modal-composition` (same feature)
- **Related pattern:** `client-bundle-safety` (same feature)

---

## Checklist Before Implementing

- [ ] Form fields don't undergo server-side transformation (slugify, trim, etc.)
- [ ] You've defined the reconstruction type explicitly (`FuncionarioBasico`, etc.)
- [ ] Reconstruction function is defined locally in Client Component (not imported from server-only lib)
- [ ] You test the reconstruction logic: compare form data → object → form display
- [ ] You document why reconstruction is preferred over re-querying (latency, UX)
