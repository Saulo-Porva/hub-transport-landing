# Supabase Quick Reference

> **Purpose**: Lookup rápido de clients, RLS, auth, erros comuns
> **MCP Validated**: 2026-06-20

## Client Types

| Client | Função | Sujeito a RLS? | Onde usar |
|--------|--------|----------------|-----------|
| `createClient()` | User context | **SIM** | Server Components, Server Actions, Route Handlers |
| `createAdminClient()` | Service role | **NÃO** | Apenas Server-side para bypass intencional de RLS |
| Browser client | Anon/User | **SIM** | Client Components (via @supabase/ssr) |

## Setup Rápido

```typescript
// lib/supabase/server.ts — SSR-safe
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'

export async function createClient() {
  const cookieStore = await cookies()
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    { cookies: { getAll: () => cookieStore.getAll(), setAll: ... } }
  )
}

export function createAdminClient() {
  return createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!,   // NUNCA NEXT_PUBLIC_
  )
}
```

## Auth — Métodos Chave

| Método | Uso | Nota |
|--------|-----|------|
| `auth.getUser()` | Validar token no servidor | ✅ Seguro — consulta o servidor Supabase |
| `auth.getSession()` | Ler sessão local | ⚠️ Não valida token — usar só no cliente para UI |
| `auth.signInWithPassword()` | Login com email/password | Em Server Action |
| `auth.signOut()` | Logout | Invalida token |
| `auth.signInWithOAuth()` | OAuth (Google, GitHub) | Redirect flow |

## Queries Comuns

```typescript
// SELECT com filtro
const { data, error } = await supabase
  .from('viagens')
  .select('id, destino, motorista_id')
  .eq('status', 'activa')
  .order('created_at', { ascending: false })

// SELECT com join
const { data } = await supabase
  .from('viagens')
  .select('*, motoristas(nome, telefone)')
  .eq('id', viagemId)
  .single()

// INSERT
const { data, error } = await supabase
  .from('viagens')
  .insert({ destino: 'Porto', motorista_id: userId })
  .select()
  .single()

// UPDATE
const { error } = await supabase
  .from('viagens')
  .update({ status: 'concluida' })
  .eq('id', viagemId)

// DELETE
const { error } = await supabase
  .from('viagens')
  .delete()
  .eq('id', viagemId)
```

## RLS — Padrões de Policy

```sql
-- Utilizador vê só os seus registos
CREATE POLICY "user_sees_own" ON viagens
  FOR SELECT USING (auth.uid() = motorista_id);

-- Admin vê tudo
CREATE POLICY "admin_sees_all" ON viagens
  FOR ALL USING (
    EXISTS (SELECT 1 FROM profiles WHERE id = auth.uid() AND role = 'admin')
  );

-- Verificar se RLS está activo
SELECT tablename, rowsecurity FROM pg_tables WHERE schemaname = 'public';

-- Listar policies de uma tabela
SELECT policyname, cmd, qual FROM pg_policies WHERE tablename = 'viagens';
```

## Erros Comuns

| Erro | Causa | Fix |
|------|-------|-----|
| `row-level security` / resultado vazio | RLS bloqueia a query | Verificar policy ou usar adminClient |
| `JWT expired` | Token expirado | Refresh automático se middleware correcto |
| `Invalid API key` | Chave errada | Confirmar ANON vs SERVICE_ROLE |
| `duplicate key` | INSERT com id já existente | Usar `.upsert()` ou verificar antes |
| `Not authenticated` em Server Action | `createClient()` sem cookies | Usar `await cookies()` no setup |

## Variáveis de Ambiente

| Var | Scope | Propósito |
|-----|-------|-----------|
| `NEXT_PUBLIC_SUPABASE_URL` | Public | URL do projecto Supabase |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Public | Chave anon (sujeita a RLS) |
| `SUPABASE_SERVICE_ROLE_KEY` | **Privada** | Bypass RLS — NUNCA NEXT_PUBLIC_ |

## See Also

- [concepts/rls-policies.md](concepts/rls-policies.md)
- [concepts/client-types.md](concepts/client-types.md)
- [patterns/debug-rls.md](patterns/debug-rls.md)
