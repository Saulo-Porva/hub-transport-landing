# Setup do Cliente Supabase em Server Components

> **Purpose**: Padrão completo e SSR-safe para usar Supabase em Next.js App Router
> **MCP Validated**: 2026-06-20

## When to Use

- Qualquer Server Component, Server Action ou Route Handler que acede ao Supabase
- Setup inicial de projecto com Supabase + Next.js

## Implementation

### Estrutura de Ficheiros

```
lib/
  supabase/
    server.ts      ← createClient() e createAdminClient()
    client.ts      ← createClient() para Client Components
    types.ts       ← Database type gerado pelo Supabase CLI (opcional)
```

### Server Client (SSR-safe)

```typescript
// lib/supabase/server.ts
import { createServerClient } from '@supabase/ssr'
import { createClient as createSupabaseClient } from '@supabase/supabase-js'
import { cookies } from 'next/headers'
import type { Database } from './types'

export async function createClient() {
  const cookieStore = await cookies()

  return createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll()
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            )
          } catch {
            // Silencioso em Server Components — middleware faz o refresh
          }
        },
      },
    }
  )
}

export function createAdminClient() {
  return createSupabaseClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!,
    {
      auth: {
        autoRefreshToken: false,
        persistSession: false,
      },
    }
  )
}
```

### Browser Client

```typescript
// lib/supabase/client.ts
import { createBrowserClient } from '@supabase/ssr'
import type { Database } from './types'

export function createClient() {
  return createBrowserClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
}
```

### Uso em Server Component

```typescript
// app/viagens/page.tsx
import { createClient } from '@/lib/supabase/server'
import { redirect } from 'next/navigation'

export default async function ViagemListPage() {
  const supabase = await createClient()

  // Verificar auth
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) redirect('/login')

  // Query com RLS automático
  const { data: viagens, error } = await supabase
    .from('viagens')
    .select('*')
    .order('created_at', { ascending: false })

  if (error) throw new Error(error.message)

  return <ViagemList viagens={viagens} />
}
```

### Uso em Server Action

```typescript
// actions/viagem.ts
'use server'
import { createClient } from '@/lib/supabase/server'
import { revalidatePath } from 'next/cache'

export async function createViagem(formData: FormData) {
  const supabase = await createClient()

  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return { error: 'Não autenticado' }

  const { error } = await supabase.from('viagens').insert({
    motorista_id: user.id,
    destino: formData.get('destino') as string,
  })

  if (error) return { error: error.message }

  revalidatePath('/viagens')
  return { success: true }
}
```

### Uso em Client Component

```typescript
// components/realtime-viagem.tsx
'use client'
import { createClient } from '@/lib/supabase/client'
import { useEffect, useState } from 'react'

export function RealtimeViagem({ id }: { id: string }) {
  const [viagem, setViagem] = useState(null)
  const supabase = createClient()

  useEffect(() => {
    const channel = supabase
      .channel(`viagem-${id}`)
      .on('postgres_changes', {
        event: 'UPDATE',
        schema: 'public',
        table: 'viagens',
        filter: `id=eq.${id}`,
      }, (payload) => {
        setViagem(payload.new)
      })
      .subscribe()

    return () => { supabase.removeChannel(channel) }
  }, [id])

  return <div>{viagem?.status}</div>
}
```

## Configuration

### Variáveis de Ambiente

```env
# .env.local
NEXT_PUBLIC_SUPABASE_URL=https://xxxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...   # NUNCA NEXT_PUBLIC_
```

## Example Usage

```typescript
// Padrão típico em Route Handler
export async function GET(request: Request) {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()

  if (!user) return new Response('Unauthorized', { status: 401 })

  const { data } = await supabase.from('viagens').select('*')
  return Response.json(data)
}
```

## See Also

- [../concepts/client-types.md](../concepts/client-types.md) — diferença entre clientes
- [../concepts/auth-ssr.md](../concepts/auth-ssr.md) — auth em App Router
- [../../nextjs/patterns/middleware.md](../../nextjs/patterns/middleware.md) — auth guard
