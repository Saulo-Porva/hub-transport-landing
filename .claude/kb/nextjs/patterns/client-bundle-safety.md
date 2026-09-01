# Pattern: Client Bundle Safety — Avoiding Server-Only Imports

> Keeping server-only packages out of Client Component bundles.

**Status:** Extracted from `MODAL_CADASTRO_EMPREGADO` (shipped 2026-08-14)

---

## Problem

You have a utility library (`src/lib/funcionario.ts`) that imports server-only packages:

```typescript
// src/lib/funcionario.ts
import postgres from 'postgres'; // Node-only! ❌

export function mensagemErroFuncionario(err: unknown): string {
  if (err instanceof postgres.PostgresError) {
    return err.message;
  }
  return "Erro desconhecido.";
}

export interface FuncionarioBasico {
  id: string;
  nome: string;
  // ...
}
```

Now a Client Component needs to import just the `FuncionarioBasico` type:

```typescript
// src/app/(app)/funcionarios/DadosBasicosTab.tsx
"use client";

import { FuncionarioBasico } from '@/lib/funcionario'; // ❌ Also pulls in postgres!
```

**Build result:**
```
✖ error TS1505: Dynamic imports are not supported when targeting 'es2015' or below and 'lib' contains 'es2015' or higher.
```

or

```
ReferenceError: net is not defined (in browser)
```

---

## Solution

**Define types locally in the Client Component; keep server-only utilities in the server library.**

### Step 1: Keep Server Utilities in Library

```typescript
// src/lib/funcionario.ts (server-only)
import postgres from 'postgres';

export function mensagemErroFuncionario(err: unknown): string {
  if (err instanceof postgres.PostgresError) {
    return err.message;
  }
  return "Erro desconhecido.";
}
```

### Step 2: Define Types in Client Component (or export from library as type-only)

**Option A: Define locally in Client Component (simplest)**

```typescript
// src/app/(app)/funcionarios/DadosBasicosTab.tsx
"use client";

export interface FuncionarioBasico {
  id: string;
  nome: string;
  cargo: string;
  setor_id: string;
  jornada_semanal_horas: number;
  grupo_regras_id: string;
  nome_social: string | null;
  matricula: number | null;
  preferencia_horario_inicio: string | null;
  preferencia_horario_fim: string | null;
  turno_preferido: "manha" | "tarde" | "indiferente" | null;
}

function funcionarioFromFormData(id: string, formData: FormData): FuncionarioBasico {
  return {
    id,
    nome: String(formData.get("nome") ?? ""),
    cargo: String(formData.get("cargo") ?? ""),
    setor_id: String(formData.get("setor_id") ?? ""),
    jornada_semanal_horas: Number(formData.get("jornada_semanal_horas") ?? 0),
    grupo_regras_id: String(formData.get("grupo_regras_id") ?? ""),
    nome_social: formData.get("nome_social") ? String(formData.get("nome_social")) : null,
    matricula: formData.get("matricula") ? Number(formData.get("matricula")) : null,
    preferencia_horario_inicio: formData.get("preferencia_horario_inicio")
      ? String(formData.get("preferencia_horario_inicio"))
      : null,
    preferencia_horario_fim: formData.get("preferencia_horario_fim")
      ? String(formData.get("preferencia_horario_fim"))
      : null,
    turno_preferido: formData.get("turno_preferido")
      ? (String(formData.get("turno_preferido")) as "manha" | "tarde" | "indiferente")
      : null,
  };
}

export function DadosBasicosTab({ funcionario, ... }: DadosBasicosTabProps) {
  // Use FuncionarioBasico, funcionarioFromFormData locally
}
```

**Option B: Export type-only from library (if you share the type)**

```typescript
// src/lib/funcionario.ts
export type FuncionarioBasico = {
  id: string;
  nome: string;
  // ...
};

// Only this is imported in Client Components:
import type { FuncionarioBasico } from '@/lib/funcionario';
// ✅ Type-only imports are tree-shaken; postgres not bundled
```

---

## Why This Works

TypeScript's `type` keyword signals to bundlers (Webpack, Turbopack, Next.js) that this import should be **erased at runtime**. Only the type definition is used at compile time; nothing is actually imported from the module.

```typescript
// ✅ Erased at runtime (safe)
import type { FuncionarioBasico } from '@/lib/funcionario';

// ❌ Imported at runtime (bundled)
import { FuncionarioBasico } from '@/lib/funcionario';
```

---

## Trade-Offs

### Pro
- ✅ Zero bundle overhead for Client Components
- ✅ Clear separation: types live near their consumers
- ✅ Compiler prevents accidental runtime imports of server modules

### Con
- ⚠️ Type duplication if multiple components need the same shape
- ⚠️ Maintainability: if the type changes, you update it in multiple places

---

## Pattern: Type-Only Re-Export

If multiple components need the same type, re-export it type-only from the library:

```typescript
// src/lib/funcionario.ts
import postgres from 'postgres'; // Server-only

export type FuncionarioBasico = {
  id: string;
  nome: string;
  // ...
};

export function mensagemErroFuncionario(err: unknown): string {
  if (err instanceof postgres.PostgresError) return err.message;
  return "Erro desconhecido.";
}
```

```typescript
// src/app/(app)/funcionarios/DadosBasicosTab.tsx
"use client";

import type { FuncionarioBasico } from '@/lib/funcionario'; // ✅ Type-only
import { useRef } from 'react';

export function DadosBasicosTab({ funcionario, ... }: { funcionario?: FuncionarioBasico; ... }) {
  // ...
}
```

```typescript
// src/app/(app)/funcionarios/FuncionarioModal.tsx
"use client";

import type { FuncionarioBasico } from '@/lib/funcionario'; // ✅ Type-only

export function FuncionarioModal({ funcionario, ... }: { funcionario?: FuncionarioBasico; ... }) {
  // ...
}
```

**Result:** Library can have server-only runtime exports (`mensagemErroFuncionario`), but types are tree-shaken away in the client bundle.

---

## Verification

### Command: `npm run build:raw`

Check that the build completes without bundler errors:

```bash
npm run build:raw
# Expected: Route /funcionarios ƒ (dynamic, OK)
#           No "TS1505: Dynamic imports" or "ReferenceError: net"
```

If you see:
```
error TS1505: Dynamic imports are not supported...
```

It means a server module was imported at runtime. Check all imports in the Client Component.

---

## Examples

### ❌ Wrong: Runtime Import of Server Module

```typescript
// src/app/(app)/funcionarios/DadosBasicosTab.tsx
"use client";

// This imports postgres into the client bundle ❌
import { FuncionarioBasico } from '@/lib/funcionario';

export function DadosBasicosTab({ funcionario }: { funcionario?: FuncionarioBasico }) {
  // ...
}
```

### ✅ Right: Type-Only Import

```typescript
// src/app/(app)/funcionarios/DadosBasicosTab.tsx
"use client";

// Type-only import; postgres stays server-side ✅
import type { FuncionarioBasico } from '@/lib/funcionario';

export function DadosBasicosTab({ funcionario }: { funcionario?: FuncionarioBasico }) {
  // ...
}
```

### ✅ Right: Define Types Locally

```typescript
// src/app/(app)/funcionarios/DadosBasicosTab.tsx
"use client";

export interface FuncionarioBasico {
  id: string;
  nome: string;
  // ...
}

export function DadosBasicosTab({ funcionario }: { funcionario?: FuncionarioBasico }) {
  // ...
}
```

---

## When to Use This Pattern

| Scenario | Use This Pattern? |
|----------|---|
| Client Component needs a type from a server-only lib | ✅ Use type-only import or define locally |
| Multiple Client Components need the same type from lib | ✅ Use type-only re-export from lib |
| Server Action uses types from `src/lib/` | ✅ No restriction; full module is allowed |
| Client Component needs a function from server-only lib | ❌ Not possible; refactor to Server Component or export differently |

---

## Common Pitfalls

### 1. Forgetting the `type` Keyword

```typescript
// ❌ Wrong
import { FuncionarioBasico } from '@/lib/funcionario';

// ✅ Right
import type { FuncionarioBasico } from '@/lib/funcionario';
```

### 2. Importing the Module for Side Effects

```typescript
// ❌ Wrong: imports the module even if you don't use anything
import '@/lib/funcionario';

// ✅ Right: type-only import if you only use types
import type { FuncionarioBasico } from '@/lib/funcionario';
```

### 3. Deep Nesting of Server Modules

If `src/lib/funcionario.ts` imports `src/lib/db.ts` which imports `postgres`, **all of it is avoided** with type-only imports:

```typescript
// src/lib/db.ts
import postgres from 'postgres'; // ← Server-only

// src/lib/funcionario.ts
import { getDb } from '@/lib/db'; // ← Server-only re-export

export type FuncionarioBasico = { ... };
export function x() { getDb(); } // ← Server function

// src/app/Cliente.tsx
import type { FuncionarioBasico } from '@/lib/funcionario'; // ← Type-only
// ✅ postgres + getDb stay out of client bundle
```

---

## References

- **Implementation:** `MODAL_CADASTRO_EMPREGADO` feature (2026-08-14)
  - `src/app/(app)/funcionarios/DadosBasicosTab.tsx` (lines 11–23, local type definition)
  - `src/lib/funcionario.ts` (server-only utilities + types)
  - `BUILD_REPORT` Deviations section (explains the decision)
- **Next.js Docs:** [Keeping server-only code out of Client Components](https://nextjs.org/docs/getting-started/react-essentials#keeping-server-only-code-out-of-the-client-bundle)
- **Related pattern:** `formdata-reconstruction` (same feature)

---

## Checklist

- [ ] Identify all server-only imports in `src/lib/` files
- [ ] Check Client Components importing from those libs
- [ ] Change imports to `type`-only where only types are used
- [ ] For shared types, confirm library exports them at top level
- [ ] Run `npm run build:raw` and verify no bundle errors
- [ ] If types are duplicated across components, consider type-only re-export pattern
