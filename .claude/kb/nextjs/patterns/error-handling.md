# Error Handling no App Router

> **Purpose**: Capturar e apresentar erros em rotas Next.js — error.tsx, not-found.tsx, global-error.tsx
> **MCP Validated**: 2026-06-20

## When to Use

- Qualquer rota que pode falhar (fetch de dados, Server Action, etc.)
- Páginas com recursos que podem não existir (404)
- Layout raiz que precisa de error boundary de último recurso

## Implementation

### error.tsx — Error Boundary de Rota

```typescript
// src/app/viagens/error.tsx
'use client'  // OBRIGATÓRIO — error.tsx deve ser Client Component

import { useEffect } from 'react'

interface ErrorProps {
  error: Error & { digest?: string }
  reset: () => void
}

export default function ViagemError({ error, reset }: ErrorProps) {
  useEffect(() => {
    // Log para serviço de erros (Sentry, etc.)
    console.error('Viagem error:', error)
  }, [error])

  return (
    <div className="flex flex-col items-center gap-4 p-8">
      <h2 className="text-lg font-semibold">Algo correu mal</h2>
      <p className="text-sm text-muted-foreground">{error.message}</p>
      <button
        onClick={reset}
        className="btn-primary"
      >
        Tentar novamente
      </button>
    </div>
  )
}
```

### not-found.tsx — 404 Customizado

```typescript
// src/app/viagens/[id]/not-found.tsx
import Link from 'next/link'

export default function ViagemNotFound() {
  return (
    <div className="p-8 text-center">
      <h2>Viagem não encontrada</h2>
      <Link href="/viagens">← Voltar às viagens</Link>
    </div>
  )
}
```

```typescript
// Invocar not-found.tsx programaticamente
import { notFound } from 'next/navigation'

export default async function ViagemPage({ params }) {
  const viagem = await getViagem(params.id)

  if (!viagem) notFound()  // renderiza not-found.tsx mais próximo

  return <ViagemDetail viagem={viagem} />
}
```

### global-error.tsx — Último Recurso

```typescript
// src/app/global-error.tsx
'use client'  // OBRIGATÓRIO

export default function GlobalError({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <html>
      <body>
        <h2>Erro crítico</h2>
        <button onClick={reset}>Recarregar</button>
      </body>
    </html>
  )
}
```

### Erros em Server Actions

```typescript
'use server'

export async function updateViagem(id: string, data: Partial<Viagem>) {
  try {
    await db.update(id, data)
    revalidatePath(`/viagens/${id}`)
  } catch (err) {
    // Retornar erro estruturado — não lançar (o cliente não vê stack traces)
    return {
      error: err instanceof Error ? err.message : 'Erro desconhecido',
    }
  }
}
```

## Configuration

| Ficheiro | Scope | Captura |
|----------|-------|---------|
| `app/error.tsx` | Todas as rotas | Erros em page.tsx e componentes filhos |
| `app/viagens/error.tsx` | Só `/viagens/*` | Erros no segmento viagens |
| `app/global-error.tsx` | Layout raiz | Erros no RootLayout |
| `app/not-found.tsx` | Global 404 | Rotas inexistentes |

## Example Usage

```typescript
// Hierarquia de error boundaries
app/
  global-error.tsx     ← captura erros no RootLayout
  layout.tsx
  error.tsx            ← captura erros nas pages raiz
  page.tsx
  viagens/
    error.tsx          ← captura só erros de /viagens/*
    layout.tsx
    page.tsx
    [id]/
      not-found.tsx    ← só para /viagens/[id] não encontrado
      page.tsx
```

## See Also

- [../concepts/server-actions.md](../concepts/server-actions.md) — erros em mutações
- [../concepts/app-router.md](../concepts/app-router.md) — file conventions
