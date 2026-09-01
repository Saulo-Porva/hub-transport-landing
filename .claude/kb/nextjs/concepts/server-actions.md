# Server Actions

> **Purpose**: Mutations seguras no servidor chamadas do client — forms, botões, operações de escrita
> **Confidence**: 0.96

## Overview

Server Actions são funções assíncronas que correm no servidor mas são invocáveis de Client Components. Substituem API Routes para mutações. Suportam `useActionState`, `useFormStatus`, e integram com revalidação de cache.

## Estrutura

```typescript
// actions/viagem.ts — ficheiro dedicado (recomendado)
'use server'

import { revalidatePath } from 'next/cache'
import { redirect } from 'next/navigation'

export async function criarViagem(prevState: unknown, formData: FormData) {
  const destino = formData.get('destino') as string

  if (!destino) {
    return { error: 'Destino é obrigatório' }
  }

  await db.viagens.insert({ destino })
  revalidatePath('/viagens')
  redirect('/viagens')
}
```

## Invocar de Client Component

```typescript
'use client'
import { useActionState } from 'react'
import { criarViagem } from '@/actions/viagem'

export function ViagemForm() {
  const [state, action, isPending] = useActionState(criarViagem, null)

  return (
    <form action={action}>
      <input name="destino" />
      {state?.error && <p>{state.error}</p>}
      <button type="submit" disabled={isPending}>
        {isPending ? 'A criar...' : 'Criar'}
      </button>
    </form>
  )
}
```

## Invocar de Server Component (sem form)

```typescript
// page.tsx (Server)
import { deleteViagem } from '@/actions/viagem'

export default function ViagemPage({ params }) {
  return (
    <form action={deleteViagem}>
      <input type="hidden" name="id" value={params.id} />
      <button type="submit">Eliminar</button>
    </form>
  )
}
```

## Invocar Programaticamente (sem form)

```typescript
'use client'
import { startTransition } from 'react'
import { updateViagem } from '@/actions/viagem'

export function SaveButton({ id, data }) {
  function handleClick() {
    startTransition(async () => {
      await updateViagem(id, data)
    })
  }
  return <button onClick={handleClick}>Guardar</button>
}
```

## Validação com Zod

```typescript
'use server'
import { z } from 'zod'

const schema = z.object({
  destino: z.string().min(1),
  data: z.string().date(),
})

export async function criarViagem(prevState: unknown, formData: FormData) {
  const parsed = schema.safeParse(Object.fromEntries(formData))

  if (!parsed.success) {
    return { errors: parsed.error.flatten().fieldErrors }
  }

  await db.insert(parsed.data)
  revalidatePath('/viagens')
}
```

## Common Mistakes

### Wrong — redirect() dentro de try/catch

```typescript
// ❌ redirect() lança uma excepção internamente — o catch apanha-a
export async function action() {
  try {
    await db.insert(data)
    redirect('/success')  // lança NextRedirectError
  } catch (err) {
    console.error(err)  // apanha o redirect!
    return { error: 'Falhou' }
  }
}

// ✅ redirect() fora do try/catch
export async function action() {
  try {
    await db.insert(data)
  } catch (err) {
    return { error: 'Falhou' }
  }
  redirect('/success')  // só chega aqui se não houve erro
}
```

### Wrong — Server Action em Client Component directo

```typescript
// ❌ Não declarar 'use server' dentro de 'use client'
'use client'
async function action() {
  'use server'  // não funciona dentro de Client Component
}

// ✅ Importar de ficheiro separado
'use client'
import { action } from '@/actions/viagem'
```

## Related

- [../patterns/data-fetching.md](../patterns/data-fetching.md) — fetch de leitura
- [server-components.md](./server-components.md) — quando usar Server vs Client
