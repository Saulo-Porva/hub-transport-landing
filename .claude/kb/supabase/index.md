# Supabase Knowledge Base

> **Purpose**: Supabase com Next.js App Router — RLS, Auth SSR, tipos de cliente, migrations, debugging
> **MCP Validated**: 2026-06-20

## Quick Navigation

### Concepts

| File | Purpose |
|------|---------|
| [concepts/rls-policies.md](concepts/rls-policies.md) | Row Level Security — escrever, debugar, tipos de policy |
| [concepts/client-types.md](concepts/client-types.md) | createClient vs createAdminClient — quando usar cada um |
| [concepts/auth-ssr.md](concepts/auth-ssr.md) | Auth em Next.js App Router com @supabase/ssr |

### Patterns

| File | Purpose |
|------|---------|
| [patterns/server-client.md](patterns/server-client.md) | Setup do cliente SSR-safe em Server Components |
| [patterns/admin-bypass.md](patterns/admin-bypass.md) | Quando e como usar serviceRole para bypass de RLS |
| [patterns/debug-rls.md](patterns/debug-rls.md) | Protocolo de diagnóstico para queries silenciosamente vazias |

---

## Quick Reference

- [quick-reference.md](quick-reference.md) — lookup rápido de clients, políticas, erros comuns

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **RLS (Row Level Security)** | Políticas por row que controlam quem pode SELECT/INSERT/UPDATE/DELETE |
| **createClient** | Cliente com contexto de auth do utilizador — sujeito a RLS |
| **createAdminClient** | Cliente com serviceRole — bypassa RLS completamente |
| **@supabase/ssr** | Biblioteca para Supabase em contexto SSR (Next.js, SvelteKit) |
| **getUser()** | Valida token no servidor Supabase — mais seguro que getSession() |

---

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| `supabase-specialist` | todos | Debugging, RLS, auth, migrations |
| `nextjs-specialist` | patterns/server-client.md | Client setup em App Router |
