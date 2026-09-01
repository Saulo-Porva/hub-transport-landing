# Data Fetching no App Router

> **Purpose**: Padrões para buscar dados em Server Components, Client Components, e com Supabase
> **MCP Validated**: 2026-06-20

## When to Use

- Fetch de dados em Server Component (sem useState, sem useEffect)
- Fetch de dados em Client Component (com SWR ou React Query)
- Queries Supabase em Server Component
- Mutações com Server Actions + revalidação

## Server Component Data Fetching (padrão recomendado)

```typescript
// app/viagens/page.tsx — Server Component
import { createClient } from '@/lib/supabase/server'

export default async function ViagemListPage() {
  const supabase = await createClient()

  const { data: viagens, error } = await supabase
    .from('viagens')
    .select('id, motorista, destino, data_partida')
    .order('data_partida', { ascending: false })

  if (error) throw new Error(error.message)  // capturado por error.tsx

  return <ViagemList viagens={viagens} />
}
```

## Fetch Paralelo (sem waterfall)

```typescript
// ✅ Paralelo — ambos começam ao mesmo tempo
export default async function DashboardPage() {
  const [viagens, motoristas] = await Promise.all([
    getViagens(),
    getMotoristas(),
  ])

  return <Dashboard viagens={viagens} motoristas={motoristas} />
}

// ❌ Sequencial (waterfall) — evitar
export default async function DashboardPage() {
  const viagens = await getViagens()       // espera
  const motoristas = await getMotoristas() // só começa depois
  ...
}
```

## Streaming com Suspense

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react'
import { ViagemListSkeleton } from '@/components/skeletons'

export default function DashboardPage() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Suspense fallback={<ViagemListSkeleton />}>
        <ViagemListServer />   {/* async Server Component */}
      </Suspense>
    </div>
  )
}

// Renderiza imediatamente o h1; ViagemList faz stream quando pronto
async function ViagemListServer() {
  const viagens = await getViagens()  // pode ser lento — não bloqueia o resto
  return <ViagemList viagens={viagens} />
}
```

## Client-Side Fetching (com SWR)

Para dados que mudam frequentemente sem navegação:

```typescript
'use client'
import useSWR from 'swr'

const fetcher = (url: string) => fetch(url).then(r => r.json())

export function ViagemStatusWidget({ id }: { id: string }) {
  const { data, error, isLoading } = useSWR(`/api/viagens/${id}/status`, fetcher, {
    refreshInterval: 5000,  // poll cada 5s
  })

  if (isLoading) return <Skeleton />
  if (error) return <ErrorMessage />

  return <StatusBadge status={data.status} />
}
```

## Padrão com Cache Tags (revalidação por domínio)

```typescript
// lib/data/viagens.ts
export async function getViagens() {
  const res = await fetch(`${process.env.API_URL}/viagens`, {
    next: { tags: ['viagens'] }
  })
  return res.json()
}

// actions/viagem.ts
'use server'
import { revalidateTag } from 'next/cache'

export async function updateViagem(id: string, data: Partial<Viagem>) {
  await db.update(id, data)
  revalidateTag('viagens')  // invalida todos os fetches com tag 'viagens'
}
```

## Fetch com Supabase em Route Handler

```typescript
// app/api/viagens/[id]/route.ts
import { createClient } from '@/lib/supabase/server'
import { NextResponse } from 'next/server'

export async function GET(
  request: Request,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params
  const supabase = await createClient()

  const { data, error } = await supabase
    .from('viagens')
    .select('*')
    .eq('id', id)
    .single()

  if (error) return NextResponse.json({ error: error.message }, { status: 404 })

  return NextResponse.json(data)
}
```

## Mutation + Revalidation (Server Action)

```typescript
'use server'
import { createClient } from '@/lib/supabase/server'
import { revalidatePath } from 'next/cache'

export async function createViagem(formData: FormData) {
  const supabase = await createClient()

  const { error } = await supabase.from('viagens').insert({
    motorista: formData.get('motorista') as string,
    destino: formData.get('destino') as string,
  })

  if (error) return { error: error.message }

  revalidatePath('/viagens')
  return { success: true }
}
```

## Anti-patterns

```typescript
// ❌ useEffect para fetch inicial — usar Server Component
'use client'
export function ViagemList() {
  const [viagens, setViagens] = useState([])
  useEffect(() => {
    fetch('/api/viagens').then(r => r.json()).then(setViagens)
  }, [])
  ...
}

// ✅ Server Component directo
export default async function ViagemList() {
  const viagens = await getViagens()
  return <ul>{viagens.map(v => <li key={v.id}>{v.destino}</li>)}</ul>
}

// ❌ Fetch em Client Component que podia ser Server
// ❌ Waterfall de fetches sequenciais — usar Promise.all
// ❌ Não chamar revalidatePath após mutação
```

## See Also

- [../concepts/caching.md](../concepts/caching.md) — estratégias de cache
- [../concepts/server-actions.md](../concepts/server-actions.md) — mutations
- [../../supabase/patterns/server-client.md](../../supabase/patterns/server-client.md) — setup do cliente
