# RLS Policies

> Row Level Security restricts which rows each database role can read or write — the primary cause of silent empty results in Supabase.

## O que é

RLS é uma feature do PostgreSQL ativada por tabela. Quando habilitado, todas as queries são filtradas pelas policies definidas — incluindo queries do SDK. Se nenhuma policy permitir acesso ao role atual (`anon`, `authenticated`, `service_role`), a query retorna `[]` **sem erro**.

`service_role` bypass RLS automaticamente. `anon` e `authenticated` são filtrados por policies.

## Quando usar

- Toda tabela que contém dados de usuário deve ter RLS habilitado
- Desabilitar RLS só em tabelas de lookup público sem dados sensíveis (ex: `countries`)
- Usar service_role client apenas server-side para operações admin

## Sintaxe / API

```sql
-- 1. Habilitar (obrigatório antes de criar policies)
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;

-- 2. Policy de leitura (USING = filtro no SELECT)
CREATE POLICY "authenticated_read_own_profile"
  ON public.profiles
  FOR SELECT
  TO authenticated
  USING (auth.uid() = id);

-- 3. Policy de escrita (WITH CHECK = validação no INSERT/UPDATE)
CREATE POLICY "authenticated_insert_own_profile"
  ON public.profiles
  FOR INSERT
  TO authenticated
  WITH CHECK (auth.uid() = id);

-- 4. Policy combinada UPDATE (USING filtra quais linhas, WITH CHECK valida novos dados)
CREATE POLICY "authenticated_update_own_profile"
  ON public.profiles
  FOR UPDATE
  TO authenticated
  USING (auth.uid() = id)
  WITH CHECK (auth.uid() = id);

-- 5. DELETE
CREATE POLICY "authenticated_delete_own"
  ON public.profiles
  FOR DELETE
  TO authenticated
  USING (auth.uid() = id);

-- 6. Acesso público (anon pode ler)
CREATE POLICY "public_read_products"
  ON public.products
  FOR SELECT
  TO anon, authenticated
  USING (true);

-- 7. Policy baseada em foreign key (ex: items de um pedido do usuário)
CREATE POLICY "authenticated_read_order_items"
  ON public.order_items
  FOR SELECT
  TO authenticated
  USING (
    EXISTS (
      SELECT 1 FROM public.orders
      WHERE orders.id = order_items.order_id
        AND orders.user_id = auth.uid()
    )
  );

-- 8. Funções úteis nas policies
auth.uid()        -- uuid do usuário autenticado
auth.role()       -- 'anon' | 'authenticated' | 'service_role'
auth.jwt()        -- jsonb com payload completo do JWT
auth.jwt() ->> 'email'          -- campo customizado do JWT
(auth.jwt() -> 'app_metadata' ->> 'role')  -- role customizado
```

## SSR Auth — Next.js App Router

```typescript
// lib/supabase/server.ts
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'

export async function createClient() {
  const cookieStore = await cookies()
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => cookieStore.getAll(),
        setAll: (cookiesToSet) => {
          cookiesToSet.forEach(({ name, value, options }) =>
            cookieStore.set(name, value, options))
        },
      },
    }
  )
}

// Server Action example
'use server'
export async function getProfile() {
  const supabase = await createClient()
  const { data, error } = await supabase
    .from('profiles')
    .select('*')
    .single()
  if (error) throw error
  return data
}
```

## Armadilhas comuns

- Criar policy sem chamar `ENABLE ROW LEVEL SECURITY` primeiro — policies não têm efeito
- Esquecer `WITH CHECK` no INSERT — política incompleta permite inserção com user_id errado
- Usar `createClient` com service_role no client-side — key vazada no bundle
- `FOR ALL` cria uma policy para todos os verbos — cuidado, pode ser excessivamente permissivo
- Não adicionar `TO authenticated` explicitamente — policy aplica a todos os roles por padrão
