# Multi-tenant RLS

> Isolamento de dados por organização usando RLS — cada usuário vê apenas dados da sua org.

## Problema

Aplicações multi-tenant precisam garantir que User A nunca veja dados da Org B, mesmo que compartilhem a mesma tabela. Fazer esse isolamento na camada de aplicação é frágil — um bug expõe dados de outros clientes.

## Solução

```sql
-- Estrutura de tabelas
CREATE TABLE public.organizations (
  id      uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name    text NOT NULL
);

CREATE TABLE public.memberships (
  user_id uuid REFERENCES auth.users NOT NULL,
  org_id  uuid REFERENCES public.organizations NOT NULL,
  role    text NOT NULL DEFAULT 'member',  -- 'admin' | 'member'
  PRIMARY KEY (user_id, org_id)
);

CREATE TABLE public.projects (
  id      uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id  uuid REFERENCES public.organizations NOT NULL,
  name    text NOT NULL,
  created_at timestamptz DEFAULT now()
);

-- Função auxiliar (executa como SECURITY DEFINER para acessar memberships)
CREATE OR REPLACE FUNCTION public.user_org_ids()
RETURNS TABLE (org_id uuid)
LANGUAGE sql STABLE SECURITY DEFINER AS $$
  SELECT org_id FROM public.memberships WHERE user_id = auth.uid();
$$;

-- RLS nas tabelas
ALTER TABLE public.projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.organizations ENABLE ROW LEVEL SECURITY;

-- Usuário vê projetos das orgs onde é membro
CREATE POLICY "members_select_projects"
  ON public.projects FOR SELECT
  TO authenticated
  USING (org_id IN (SELECT org_id FROM public.user_org_ids()));

-- Apenas admins criam projetos
CREATE POLICY "admins_insert_projects"
  ON public.projects FOR INSERT
  TO authenticated
  WITH CHECK (
    EXISTS (
      SELECT 1 FROM public.memberships
      WHERE user_id = auth.uid()
        AND org_id = projects.org_id
        AND role = 'admin'
    )
  );

-- Usuário vê suas próprias orgs
CREATE POLICY "members_select_orgs"
  ON public.organizations FOR SELECT
  TO authenticated
  USING (id IN (SELECT org_id FROM public.user_org_ids()));
```

```typescript
// Client-side: query normal — RLS filtra automaticamente
const { data: projects } = await supabase
  .from('projects')
  .select('id, name, org_id')
// Retorna apenas projetos das orgs do usuário autenticado

// Admin server-side: bypass RLS para operações de suporte
const adminSupabase = createClient(url, serviceRoleKey)
const { data: allProjects } = await adminSupabase
  .from('projects')
  .select('*')
  .eq('org_id', orgId)
```

## Variações

**Sem função auxiliar** — usar subquery inline (mais simples, menos reutilizável):
```sql
USING (org_id IN (
  SELECT org_id FROM public.memberships WHERE user_id = auth.uid()
))
```

**JWT custom claims** — armazenar `org_ids` no JWT para eliminar join com memberships em cada query (requer Supabase Auth Hook):
```sql
USING (org_id = ANY((auth.jwt() -> 'app_metadata' ->> 'org_ids')::uuid[]))
```

**Row-level audit** — adicionar `created_by uuid DEFAULT auth.uid()` e policy `USING (created_by = auth.uid())` para isolamento por usuário dentro da org.
