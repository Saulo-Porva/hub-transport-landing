# Client Types — createClient vs createAdminClient

> **Purpose**: Diferença entre clientes Supabase, quando usar cada um, e como configurar SSR-safe
> **MCP Validated**: 2026-06-20

## Regra de Ouro

```
createClient()       → contexto do utilizador → sujeito a RLS
createAdminClient()  → serviceRole           → bypassa RLS completamente
```

**Nunca use `createAdminClient()` em Client Components.** A `SUPABASE_SERVICE_ROLE_KEY` exposta no browser é uma falha crítica de segurança.

## Comparação

| Aspecto | createClient | createAdminClient |
|---------|--------------|-------------------|
| Chave usada | `NEXT_PUBLIC_SUPABASE_ANON_KEY` | `SUPABASE_SERVICE_ROLE_KEY` |
| Contexto de auth | Utilizador autenticado (via cookies) | Nenhum — bypass total |
| RLS | Aplicado | Bypassado |
| Onde usar | Server Components, Actions, Handlers, Client | Apenas server-side |
| Quando usar | 99% dos casos | Tarefas admin, cron jobs, exportações |

## Setup em Next.js App Router

```typescript
// lib/supabase/server.ts
import { createServerClient } from '@supabase/ssr'
import { createClient as createSupabaseClient } from '@supabase/supabase-js'
import { cookies } from 'next/headers'
import type { Database } from '@/lib/supabase/types'

// SSR-safe client — carrega cookies do request
export async function createClient() {
  const cookieStore = await cookies()  // Next 15: await obrigatório

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
            // Ignorar em Server Components — middleware actualiza os cookies
          }
        },
      },
    }
  )
}

// Admin client — bypass de RLS (só server-side)
export function createAdminClient() {
  return createSupabaseClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!,  // NUNCA NEXT_PUBLIC_
    {
      auth: {
        autoRefreshToken: false,
        persistSession: false,
      },
    }
  )
}
```

## Client Component Setup

```typescript
// lib/supabase/client.ts — para Client Components
import { createBrowserClient } from '@supabase/ssr'
import type { Database } from '@/lib/supabase/types'

export function createClient() {
  return createBrowserClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
}
```

## Quando Usar createAdminClient

| Caso de uso | Razão |
|-------------|-------|
| Exportação de dados (admin vê tudo) | RLS impediria dados de outros utilizadores |
| Cron jobs e tarefas de sistema | Sem utilizador autenticado |
| Criar utilizadores programaticamente | `auth.admin.createUser()` |
| Sincronização de dados entre tabelas | Sem contexto de user session |
| Backfill e migrações de dados | Operações de sistema, não de utilizador |

## Anti-patterns

```typescript
// ❌ adminClient em Client Component
'use client'
import { createAdminClient } from '@/lib/supabase/server'

export function DangerousComponent() {
  // createAdminClient importa SUPABASE_SERVICE_ROLE_KEY
  // que não existe no browser — vai falhar em produção
  // e mesmo que existisse, seria uma falha de segurança
  const admin = createAdminClient()
}

// ❌ NEXT_PUBLIC_ na service role key
// .env.local
NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY=eyJ...  // ERRADO — expõe ao browser

// ✅ Privada
SUPABASE_SERVICE_ROLE_KEY=eyJ...              // Correcto — só servidor

// ❌ Usar adminClient para operações de utilizador comum
export async function getMinhasViagens() {
  const admin = createAdminClient()
  // Bypassa RLS — utilizador vê viagens de outros!
  return admin.from('viagens').select('*')
}

// ✅ createClient com contexto do utilizador
export async function getMinhasViagens() {
  const supabase = await createClient()
  // RLS aplica — utilizador só vê as suas
  return supabase.from('viagens').select('*')
}
```

## Diagnóstico Rápido

```
Dado retorna vazio com createClient → RLS a filtrar → verificar policy
Dado retorna OK com createAdminClient → confirma que é problema de RLS
Erro "JWT" com createClient → cookie não carregado → verificar middleware
Erro "Invalid API key" → chave errada ou variável de ambiente não definida
```

## See Also

- [rls-policies.md](./rls-policies.md) — escrever e debugar policies
- [auth-ssr.md](./auth-ssr.md) — auth em Next.js App Router
- [../patterns/admin-bypass.md](../patterns/admin-bypass.md) — quando e como usar adminClient
