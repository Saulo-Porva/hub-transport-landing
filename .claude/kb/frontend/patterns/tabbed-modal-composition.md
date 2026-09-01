# Pattern: Tabbed Modal Composition

> Consolidating multiple independent forms into a single modal with tabs, keeping business logic intact.

**Status:** Extracted from `MODAL_CADASTRO_EMPREGADO` (shipped 2026-08-14)

---

## Problem

You have multiple related forms/actions that live in separate modals (e.g., `NovoFuncionarioForm`, `EditarFuncionarioForm`, `RestricoesForm`, `SetoresSecundariosForm`). UX requests consolidating them into a single modal with tabs to reduce friction and enable multi-step workflows (e.g., create employee, then add restrictions in the same session).

**Constraints:**
- Cannot refactor the existing forms' business logic (already tested, in production)
- Cannot modify shared UI components (e.g., `Tabs.tsx`) if they're used elsewhere
- Some tabs may not make sense in all modes (e.g., Restrições tab requires `funcionario_id`)

---

## Solution

### Architecture

```
FuncionarioModal (shell, mode state)
├── DadosBasicosTab (new, extracted from 2 forms)
├── RestricoesTab (new, extracted from 1 form)
└── SetoresSecundariosTab (new, extracted from 1 form)

plus Tabs.tsx (unmodified, shared component)
```

### Step 1: Create Tab Content Components

Extract the business logic (form fields, Server Action calls, success handlers) from each existing form into tab-specific components:

```typescript
// Before: NovoFuncionarioForm.tsx + EditarFuncionarioForm.tsx (2 components, duplication)
// After: DadosBasicosTab.tsx (1 component, shared by both flows)

export interface DadosBasicosTabProps {
  funcionario?: FuncionarioBasico; // undefined = criação, present = edição
  action: (prevState, formData) => Promise<ActionResult>;
  onSuccess: (updated: FuncionarioBasico) => void;
}

export function DadosBasicosTab({ funcionario, action, onSuccess }: DadosBasicosTabProps) {
  const formRef = useRef<HTMLFormElement>(null);
  
  function handleSuccess(state: ActionResult<{ id: string }>) {
    const id = funcionario?.id ?? state.data?.id;
    // Reconstruct object from FormData to avoid N+1 query
    onSuccess(funcionarioFromFormData(id, new FormData(formRef.current)));
  }

  return (
    <ServerActionForm action={action} formRef={formRef} onSuccess={handleSuccess}>
      {/* form fields */}
    </ServerActionForm>
  );
}
```

### Step 2: Build the Shell Component

Create a wrapper that manages modal state and conditionally renders tabs:

```typescript
export interface FuncionarioModalProps {
  modo: "criar" | "editar";
  funcionario?: FuncionarioBasico;
  criarAction?: FuncionarioAction;
  editarAction: FuncionarioAction;
  trigger: React.ReactNode;
}

export function FuncionarioModal({
  modo,
  funcionario,
  criarAction,
  editarAction,
  trigger,
}: FuncionarioModalProps) {
  const [open, setOpen] = useState(false);
  const [modoAtual, setModoAtual] = useState<"criar" | "editar">(modo);
  const [funcionarioAtual, setFuncionarioAtual] = useState<FuncionarioBasico | undefined>(
    funcionario,
  );

  const tabs = [
    {
      key: "dados",
      label: "Dados básicos",
      content: (
        <DadosBasicosTab
          funcionario={funcionarioAtual}
          action={modoAtual === "criar" ? criarAction : editarAction}
          onSuccess={(atualizado) => {
            setFuncionarioAtual(atualizado);
            if (modoAtual === "criar") {
              setModoAtual("editar"); // Transition, don't close
            } else {
              setOpen(false); // Close after edit
            }
          }}
        />
      ),
    },
  ];

  // Conditionally add tabs based on mode
  if (modoAtual === "editar" && funcionarioAtual) {
    tabs.push(
      { key: "restricoes", label: "Restrições", content: <RestricoesTab {...} /> },
      { key: "setores", label: "Setores", content: <SetoresSecundariosTab {...} /> },
    );
  }

  return (
    <>
      <span onClick={() => setOpen(true)}>{trigger}</span>
      {open && (
        <div role="dialog" className="...">
          <Tabs tabs={tabs} defaultTab="dados" />
        </div>
      )}
    </>
  );
}
```

### Step 3: Conditional Tab Rendering (Key Decision)

Instead of adding a `disabled` prop to `Tabs.tsx`, simply don't include tabs for modes where they don't apply:

```typescript
// ✅ Good: conditional array
const tabs = [{ key: "dados", ... }];
if (modoAtual === "editar") {
  tabs.push({ key: "restricoes", ... });
}

// ❌ Avoid: disabling in shared component
// Don't add disabled={modoAtual !== "editar"} to Tabs.tsx
```

**Why:** Avoids modifying shared UI components for feature-specific logic. Keeps concerns separated.

---

## Benefits

| Benefit | Why |
|---------|-----|
| **Code reuse** | Tab content components inherit existing form logic (no duplication) |
| **State management** | Shell component owns mode transitions (criação→edição in one session) |
| **Shared component safety** | No need to modify `Tabs.tsx` for this feature |
| **Clear API** | `modo` + `criarAction`/`editarAction` make workflow intent explicit |

---

## Gotchas

### 1. Mode-Based Action Switching

During criação→edição transition, switch which Server Action is called:

```typescript
// Wrong: always call criarAction
const action = criarAction;

// Right: switch based on mode
const action = modoAtual === "criar" ? criarAction : editarAction;
```

If you don't switch, editing the "Dados básicos" tab after creation will call `criarFuncionario` again (duplicate employee).

### 2. Uncontrolled Inputs + Tab Unmounting

If tab content uses `<input defaultValue={...}>` (uncontrolled), **switching tabs will lose unsaved edits**:

```typescript
// Tabs.tsx unmounts inactive tab content:
{tabs.find((t) => t.key === active)?.content}

// On remount, DadosBasicosTab re-renders with defaultValue from props
// Any user edits (stored in DOM, not state) are lost
```

**Mitigations (pick one):**
- Accept the limitation for v1 (OK if data is auto-saved per item, like Restrições)
- Switch to controlled inputs (state lifting cost)
- Keep all tabs mounted (performance cost)

See `MODAL_CADASTRO_EMPREGADO` SHIPPED document for full discussion.

---

## Applicability

**Use this pattern when:**
- Consolidating 2–4 related forms into a modal
- Forms don't share state (each has its own Server Action)
- You want to avoid modifying shared UI components
- State transitions (e.g., criação→edição) are part of the workflow

**Avoid if:**
- Forms need deeply shared client-side state (lift to parent instead)
- You're willing to modify the shared `Tabs` component for a feature-specific need

---

## References

- **Implementation:** `MODAL_CADASTRO_EMPREGADO` feature (2026-08-14)
  - `src/app/(app)/funcionarios/FuncionarioModal.tsx` (shell)
  - `src/app/(app)/funcionarios/DadosBasicosTab.tsx` (tab content)
  - `src/components/ui/Tabs.tsx` (unmodified shared component)
- **Related pattern:** `formdata-reconstruction` (same feature)
- **Related pattern:** `client-bundle-safety` (same feature)

---

## Questions to Ask Before Using

1. **Are all existing forms' Server Actions still used directly elsewhere?** (If yes, confirm before removing their top-level imports)
2. **Does your modal need to handle state transitions between tabs?** (If yes, this pattern is ideal)
3. **Can you accept uncontrolled input behavior (data loss on tab switch)?** (If no, plan for controlled inputs or persistent-mount tabs)
