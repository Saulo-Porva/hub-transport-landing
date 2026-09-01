# Debugging RLS — Protocolo de Diagnóstico

> **Purpose**: Resolver queries que retornam vazio ou erros inesperados por causa de RLS
> **MCP Validated**: 2026-06-20

## Sintomas de Problema de RLS

- Query retorna `[]` quando devia ter dados, com `error: null`
- INSERT falha com "new row violates row-level security policy"
- UPDATE/DELETE sem efeito (afectou 0 rows) e sem erro explícito
- App funciona em localhost mas não em produção (auth não configurado correctamente)

## Protocolo de Diagnóstico (5 passos)

### Passo 1 — Confirmar que RLS está activo

```sql
-- No Supabase Dashboard > SQL Editor
SELECT tablename, rowsecurity
FROM pg_tables
WHERE schemaname = 'public' AND tablename = 'viagens';
```

Se `rowsecurity = false` → RLS desactivado, problema noutro sítio.

### Passo 2 — Listar políticas existentes

```sql
SELECT policyname, cmd, qual, with_check
FROM pg_policies
WHERE schemaname = 'public' AND tablename = 'viagens';
```

Se tabela sem nenhuma policy → **nenhuma row é acessível** (RLS bloqueante por defeito).

### Passo 3 — Testar com adminClient (confirmar que dados existem)

```typescript
// Criar um endpoint de diagnóstico temporário
import { createAdminClient } from '@/lib/supabase/server'

export async function GET() {
  const admin = createAdminClient()
  const { data, count } = await admin
    .from('viagens')
    .select('*', { count: 'exact' })

  return Response.json({ count, sample: data?.slice(0, 3) })
}
```

Se adminClient retorna dados mas createClient() retorna `[]` → **confirmado problema de RLS**.

### Passo 4 — Verificar o user ID na sessão

```typescript
// Em Server Action ou Route Handler
export async function debugAuth() {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()

  console.log('User ID:', user?.id)
  console.log('User metadata:', user?.user_metadata)

  // Verificar se este ID existe na tabela
  const admin = createAdminClient()
  const { data } = await admin
    .from('viagens')
    .select('id, motorista_id')
    .eq('motorista_id', user?.id ?? '')
    .limit(5)

  console.log('Rows matching user:', data)
}
```

### Passo 5 — Simular no SQL Editor como utilizador específico

```sql
-- Substituir UUID_DO_USER pelo id do utilizador
-- No Supabase Dashboard > SQL Editor
SET LOCAL role TO authenticated;
SET LOCAL "request.jwt.claims" TO '{"sub": "UUID_DO_USER", "role": "authenticated"}';

SELECT * FROM viagens;
-- Se retorna vazio mas o user tem rows → policy errada
-- Se retorna rows → problema no código da app, não em RLS
```

## Soluções Comuns

### Tabela sem nenhuma policy (bloqueia tudo)

```sql
-- Adicionar policy básica
CREATE POLICY "user_sees_own" ON viagens
  FOR SELECT
  USING (auth.uid() = motorista_id);
```

### motorista_id não corresponde a auth.uid()

```sql
-- Verificar tipo dos campos
SELECT pg_typeof(id) FROM auth.users LIMIT 1;  -- uuid
SELECT pg_typeof(motorista_id) FROM viagens LIMIT 1;  -- deve ser uuid

-- Comparar com cast explícito se necessário
USING (motorista_id = auth.uid()::uuid)
```

### Policy com subquery recursiva

```sql
-- ❌ Problema: SELECT na mesma tabela dentro de USING
CREATE POLICY "bad_policy" ON viagens
  FOR SELECT USING (
    id IN (SELECT id FROM viagens WHERE motorista_id = auth.uid())
  );

-- ✅ Fix: referência directa
CREATE POLICY "good_policy" ON viagens
  FOR SELECT USING (motorista_id = auth.uid());
```

### Cookie de sessão não carregado (middleware ausente)

```typescript
// Sintoma: user = null em Server Component mesmo com utilizador autenticado no browser
// Causa: middleware.ts não existe ou o matcher não abrange a rota

// Fix: confirmar que middleware.ts existe e o matcher cobre a rota problemática
// Ver kb/nextjs/patterns/middleware.md para o padrão completo
```

## Checklist de Debug Rápido

```
□ Dados existem no dashboard (SQL Editor sem RLS)?
□ RLS está activo na tabela (`rowsecurity = true`)?
□ Existem policies para a operação (SELECT/INSERT/UPDATE)?
□ O auth.uid() na sessão corresponde ao campo da policy?
□ O middleware está a correr e a carregar os cookies?
□ Está a usar createClient() (não createAdminClient()) para operações de utilizador?
```

## See Also

- [../concepts/rls-policies.md](../concepts/rls-policies.md) — escrever policies correctas
- [../concepts/client-types.md](../concepts/client-types.md) — createClient vs createAdminClient
- [../../nextjs/patterns/middleware.md](../../nextjs/patterns/middleware.md) — middleware auth
