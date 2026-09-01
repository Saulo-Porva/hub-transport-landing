# Admin Bypass — Quando e Como Usar createAdminClient

> **Purpose**: Casos legítimos para bypass de RLS e como fazê-lo com segurança
> **MCP Validated**: 2026-06-20

## Quando Usar

O `createAdminClient()` bypassa RLS completamente. Usar apenas quando:

| Caso | Razão |
|------|-------|
| Exportação de dados por admin | Admin vê dados de todos os utilizadores |
| Cron jobs / tarefas agendadas | Sem utilizador autenticado |
| Criar/gerir utilizadores | `auth.admin.*` requer serviceRole |
| Backfill e migrações | Operações de sistema |
| Verificar existência antes de RLS ser aplicado | Lógica condicional de negócio |
| Sincronização entre tabelas | Operações cross-user |

## Implementation

```typescript
// Exportação de dados — admin vê tudo
import { createAdminClient } from '@/lib/supabase/server'

export async function exportAllViagens(adminUserId: string) {
  // Verificar primeiro se o utilizador é admin com createClient()
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()

  if (!user || user.user_metadata?.role !== 'admin') {
    throw new Error('Forbidden')
  }

  // Só agora usar adminClient
  const adminClient = createAdminClient()
  const { data: viagens } = await adminClient
    .from('viagens')
    .select('*')
    .gte('created_at', startDate)
    .lte('created_at', endDate)

  return viagens
}
```

```typescript
// Criar utilizador programaticamente
export async function createMotorista(email: string, nome: string) {
  const adminClient = createAdminClient()

  const { data: { user }, error } = await adminClient.auth.admin.createUser({
    email,
    password: generateTemporaryPassword(),
    user_metadata: {
      nome,
      role: 'motorista',
    },
    email_confirm: true,
  })

  if (error) throw error

  // Criar perfil na tabela profiles
  await adminClient.from('profiles').insert({
    id: user!.id,
    nome,
    role: 'motorista',
  })

  return user
}
```

```typescript
// Cron job — sem contexto de utilizador
export async function dailyCleanup() {
  const adminClient = createAdminClient()

  // Fechar viagens activas com mais de 24h (nenhum utilizador autenticado aqui)
  const { error } = await adminClient
    .from('viagens')
    .update({ status: 'expirada' })
    .eq('status', 'activa')
    .lt('created_at', new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString())

  if (error) console.error('Cleanup failed:', error)
}
```

## Segurança — Checklist

Antes de usar `createAdminClient()`:

- [ ] A operação é genuinamente de sistema (não de utilizador)?
- [ ] Se envolve um utilizador: verifiquei o seu role com `createClient()` primeiro?
- [ ] O `createAdminClient()` está num ficheiro server-side (`.ts` sem `'use client'`)?
- [ ] A `SUPABASE_SERVICE_ROLE_KEY` não está prefixada com `NEXT_PUBLIC_`?
- [ ] O resultado não expõe dados de outros utilizadores ao requester errado?

## Anti-patterns

```typescript
// ❌ Usar adminClient para queries normais de utilizador
export async function getMinhasViagens() {
  const admin = createAdminClient()
  // RLS bypassado — utilizador vê TUDO!
  return admin.from('viagens').select('*')
}

// ❌ adminClient em Client Component
'use client'
const admin = createAdminClient()  // SUPABASE_SERVICE_ROLE_KEY não existe no browser

// ❌ Verificar auth com adminClient
const admin = createAdminClient()
const { data: { user } } = await admin.auth.getUser()
// Isto valida o token mas perde o contexto de RLS — usar createClient() para auth

// ✅ Padrão correcto: createClient() para auth, adminClient() para operação
export async function exportData() {
  // Passo 1: verificar auth com user client
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user || user.user_metadata?.role !== 'admin') throw new Error('Forbidden')

  // Passo 2: só então usar adminClient
  const adminClient = createAdminClient()
  return adminClient.from('viagens').select('*')
}
```

## See Also

- [../concepts/client-types.md](../concepts/client-types.md) — diferença entre clientes
- [../concepts/rls-policies.md](../concepts/rls-policies.md) — como RLS funciona
- [../patterns/debug-rls.md](../patterns/debug-rls.md) — debugar quando RLS bloqueia inesperadamente
