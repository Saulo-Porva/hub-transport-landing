# Row Level Security (RLS) — Políticas e Debugging

> **Purpose**: Escrever, debugar e entender RLS no Supabase
> **MCP Validated**: 2026-06-20

## Overview

RLS filtra rows automaticamente com base no utilizador autenticado. Uma query que retorna `[]` quando deveria retornar dados é quase sempre um problema de RLS — não um bug de código.

**Regra de ouro:** `createClient()` está sujeito a RLS. `createAdminClient()` (serviceRole) bypassa RLS.

## Activar RLS

```sql
-- RLS está DESACTIVADO por defeito nas tabelas novas
ALTER TABLE viagens ENABLE ROW LEVEL SECURITY;

-- Verificar estado de todas as tabelas
SELECT tablename, rowsecurity
FROM pg_tables
WHERE schemaname = 'public';
```

## Tipos de Policy

### SELECT — Quem pode ler

```sql
-- Utilizador vê só os seus registos
CREATE POLICY "motorista_vê_próprias_viagens" ON viagens
  FOR SELECT
  USING (auth.uid() = motorista_id);

-- Admin vê tudo
CREATE POLICY "admin_vê_tudo" ON viagens
  FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM profiles
      WHERE id = auth.uid() AND role = 'admin'
    )
  );

-- Qualquer autenticado vê
CREATE POLICY "autenticados_vêem" ON viagens
  FOR SELECT
  USING (auth.role() = 'authenticated');
```

### INSERT — Quem pode criar

```sql
-- Utilizador cria viagens para si próprio
CREATE POLICY "motorista_cria_próprias" ON viagens
  FOR INSERT
  WITH CHECK (auth.uid() = motorista_id);
```

### UPDATE — Quem pode editar

```sql
-- Só o dono pode editar, e só o próprio registo
CREATE POLICY "motorista_edita_próprias" ON viagens
  FOR UPDATE
  USING (auth.uid() = motorista_id)
  WITH CHECK (auth.uid() = motorista_id);
```

### DELETE — Quem pode apagar

```sql
CREATE POLICY "motorista_apaga_próprias" ON viagens
  FOR DELETE
  USING (auth.uid() = motorista_id);
```

### Policy para Todas as Operações

```sql
CREATE POLICY "admin_total_access" ON viagens
  FOR ALL
  USING (
    EXISTS (SELECT 1 FROM profiles WHERE id = auth.uid() AND role = 'admin')
  )
  WITH CHECK (
    EXISTS (SELECT 1 FROM profiles WHERE id = auth.uid() AND role = 'admin')
  );
```

## Debugging RLS

### 1. Verificar se RLS está activo

```sql
SELECT tablename, rowsecurity
FROM pg_tables
WHERE schemaname = 'public' AND tablename = 'viagens';
-- rowsecurity = true → RLS activo
```

### 2. Listar policies existentes

```sql
SELECT policyname, cmd, qual, with_check
FROM pg_policies
WHERE tablename = 'viagens';
```

### 3. Simular como utilizador específico

```sql
-- No SQL Editor do Supabase Dashboard
SET LOCAL role TO authenticated;
SET LOCAL "request.jwt.claims" TO '{"sub": "UUID_DO_UTILIZADOR"}';
SELECT * FROM viagens;  -- vê o que o utilizador veria
```

### 4. Query sem RLS (Supabase Dashboard)

O SQL Editor do dashboard corre como `postgres` (superuser) — bypassa RLS. Se vês dados no dashboard mas não na app, é problema de RLS.

## Erros Comuns de RLS

### Resultado vazio sem error

```typescript
const { data, error } = await supabase.from('viagens').select('*')
// data: []
// error: null
// → RLS está a filtrar todas as rows
```

**Diagnóstico:** Verificar se `auth.uid()` corresponde ao `motorista_id` nas rows.

### Policy em loop / recursão

```sql
-- ❌ NUNCA fazer SELECT na mesma tabela dentro de USING
CREATE POLICY "bad" ON viagens
  FOR SELECT USING (
    id IN (SELECT id FROM viagens WHERE motorista_id = auth.uid())  -- loop!
  );

-- ✅ Referência directa
CREATE POLICY "good" ON viagens
  FOR SELECT USING (motorista_id = auth.uid());
```

### Policy com TRUE (inseguro)

```sql
-- ❌ Qualquer um pode ver tudo — só para debug
CREATE POLICY "debug_all" ON viagens
  FOR SELECT USING (true);
-- Remover antes de produção!
```

## Testar RLS na Aplicação

```typescript
// Testar se utilizador A não vê registos do utilizador B
const supabaseA = createClientWithUser(userA_token)
const supabaseB = createClientWithUser(userB_token)

const { data: dataFromA } = await supabaseA.from('viagens').select('*')
const { data: dataFromB } = await supabaseB.from('viagens').select('*')

// dataFromA deve ter só viagens de A
// dataFromB deve ter só viagens de B
// nenhum deve ver viagens do outro
```

## See Also

- [client-types.md](./client-types.md) — createClient vs createAdminClient
- [../patterns/debug-rls.md](../patterns/debug-rls.md) — protocolo de diagnóstico passo a passo
- [../patterns/admin-bypass.md](../patterns/admin-bypass.md) — quando fazer bypass de RLS
