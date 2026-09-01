# Variáveis de Ambiente no Vercel

> **Purpose**: Scopes, NEXT_PUBLIC_, segredos, e gestão de env vars em múltiplos ambientes
> **MCP Validated**: 2026-06-20

## Como Funciona

O Vercel injjecta variáveis de ambiente em build time e em runtime, dependendo do tipo:

| Tipo | Quando injectada | Quem acede |
|------|-----------------|------------|
| `NEXT_PUBLIC_*` | **Build time** — embutida no bundle JS | Browser + Servidor |
| Sem prefixo | Runtime — disponível no servidor | Só Servidor |

**Consequência crítica:** `NEXT_PUBLIC_*` está no bundle JS do browser. Qualquer segredo com este prefixo é exposto publicamente.

## Scopes de Ambiente

```
Vercel Dashboard → Projecto → Settings → Environment Variables

┌──────────────────────────────────────────────────────┐
│ NOME_DA_VAR                                          │
│ valor: ****                                          │
│                                                      │
│ Environments:                                        │
│ ☑ Production    ← deploy de main                    │
│ ☑ Preview       ← outros branches e PRs             │
│ ☑ Development   ← vercel dev local                  │
└──────────────────────────────────────────────────────┘
```

Cada scope pode ter **valores diferentes** para a mesma variável.

## Padrão de Configuração

```
# Variáveis para TODOS os ambientes
NEXT_PUBLIC_SUPABASE_URL=https://xxxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...

# Variáveis diferentes por ambiente
DATABASE_URL:
  Production → postgres://prod-server/db
  Preview    → postgres://staging-server/db
  Development → postgres://localhost/db

# Segredos — só Production
SUPABASE_SERVICE_ROLE_KEY=eyJ...  (Production only)
STRIPE_SECRET_KEY=sk_live_...     (Production only)

# Preview usa chaves de teste
STRIPE_SECRET_KEY=sk_test_...     (Preview only)
```

## Regras de Segurança

```
✅ NEXT_PUBLIC_SUPABASE_URL         → exposto — intencionalmente público
✅ NEXT_PUBLIC_SUPABASE_ANON_KEY    → exposto — protegido por RLS

❌ NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY  → NUNCA! Bypassa RLS
❌ NEXT_PUBLIC_STRIPE_SECRET_KEY          → NUNCA! Permite cobranças
❌ NEXT_PUBLIC_DATABASE_URL               → NUNCA! Acesso directo à BD
```

## Usar env vars em código

```typescript
// Server-side (Node.js runtime)
const serviceRoleKey = process.env.SUPABASE_SERVICE_ROLE_KEY  // ✅

// Client-side / SSR
const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL  // ✅ sempre disponível

// ❌ Erro comum — variável privada no browser
'use client'
const secret = process.env.SUPABASE_SERVICE_ROLE_KEY  // undefined no browser!
```

## Puxar Env Vars para Local

```bash
# Instalar Vercel CLI
npm i -g vercel

# Link do projecto local ao Vercel
vercel link

# Puxar env vars de Development para .env.local
vercel env pull .env.local

# .env.local não deve ser commitado (.gitignore)
```

## Gestão via CLI

```bash
# Listar todas as vars
vercel env ls

# Adicionar var para Production
vercel env add STRIPE_SECRET_KEY production

# Adicionar var para Preview e Development
vercel env add STRIPE_SECRET_KEY preview development

# Remover var
vercel env rm STRIPE_SECRET_KEY

# Ver valor de uma var (truncado por segurança)
vercel env inspect STRIPE_SECRET_KEY
```

## Vars em next.config

```typescript
// next.config.ts — para vars não-NEXT_PUBLIC_ disponíveis no browser
const nextConfig = {
  env: {
    CUSTOM_VAR: process.env.CUSTOM_VAR,  // disponível em process.env no browser
  },
}
// Usar com moderação — NEXT_PUBLIC_ é mais claro e explícito
```

## Problemas Comuns

```
Problema: Var disponível em Preview mas não em Production
Fix: Verificar scope — adicionar ao scope "Production" no dashboard

Problema: NEXT_PUBLIC_ undefined em runtime
Fix: Confirmar que o valor está definido no scope correcto em Vercel.
     Vars NEXT_PUBLIC_ precisam de rebuild para tomar efeito.

Problema: Var com valor correcto no .env.local mas não na Vercel Preview
Fix: `vercel env pull` só puxa Development. Preview pode ter valor diferente.
     Verificar no dashboard: Settings → Environment Variables.

Problema: undefined no Client Component para var sem NEXT_PUBLIC_
Fix: Vars sem prefixo não existem no browser. Usar Server Component
     ou passar como prop.
```

## See Also

- [deployment-model.md](./deployment-model.md) — ambientes e runtimes
- [../patterns/debug-functions.md](../patterns/debug-functions.md) — logs quando var não está definida
- [../../supabase/concepts/client-types.md](../../supabase/concepts/client-types.md) — Supabase keys
