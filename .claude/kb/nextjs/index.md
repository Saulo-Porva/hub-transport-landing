# Next.js Knowledge Base

> **Purpose**: Next.js 15 App Router — padrões, diagnóstico e configuração para aplicações SSR/RSC
> **MCP Validated**: 2026-06-20

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/server-components.md](concepts/server-components.md) | RSC vs Client Components — quando usar cada um |
| [concepts/app-router.md](concepts/app-router.md) | File conventions, layouts, rotas dinâmicas |
| [concepts/server-actions.md](concepts/server-actions.md) | Mutations, forms, 'use server' directive |
| [concepts/caching.md](concepts/caching.md) | Cache behaviour, revalidatePath, revalidateTag |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/data-fetching.md](patterns/data-fetching.md) | Parallel fetch, streaming, Suspense |
| [patterns/error-handling.md](patterns/error-handling.md) | error.tsx, not-found.tsx, global-error.tsx |
| [patterns/middleware.md](patterns/middleware.md) | Auth guards, redirects, cookie refresh |

---

## Quick Reference

- [quick-reference.md](quick-reference.md) — lookup rápido de directives, file conventions, cache options

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **RSC (React Server Component)** | Roda no servidor, zero JS no bundle client, pode ser async |
| **Client Component** | `'use client'` no topo, roda no browser, pode usar hooks/events |
| **Server Action** | Função `'use server'` chamada do client, roda no servidor |
| **Route Handler** | `route.ts` — endpoint HTTP (substitui API Routes do Pages Router) |
| **revalidatePath** | Invalida cache de uma rota após mutação |

---

## Learning Path

| Level | Files |
|-------|-------|
| **Diagnóstico rápido** | quick-reference.md |
| **Entender RSC** | concepts/server-components.md |
| **Implementar mutações** | concepts/server-actions.md, patterns/data-fetching.md |
| **Resolver cache** | concepts/caching.md |
| **Error boundaries** | patterns/error-handling.md |

---

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| `nextjs-specialist` | todos | Debugging, diagnóstico, configuração |
| `ui-designer` | patterns/error-handling.md | Error states visuais |
| `supabase-specialist` | patterns/middleware.md | Auth middleware com Supabase |
