# App Router — Convenções e File System Routing

> **Purpose**: File conventions, rotas dinâmicas, route groups, e layout nesting no App Router
> **MCP Validated**: 2026-06-20

## Overview

O App Router usa o sistema de ficheiros como router. Cada pasta em `app/` é um segmento de rota. Os ficheiros especiais (`page.tsx`, `layout.tsx`, etc.) definem o comportamento de cada segmento.

## File Conventions

| Ficheiro | Propósito | Obrigatório? |
|----------|-----------|--------------|
| `page.tsx` | UI única para a rota — torna o segmento acessível | Sim |
| `layout.tsx` | UI partilhada entre `page.tsx` filhos | Não |
| `loading.tsx` | Suspense boundary automático enquanto page carrega | Não |
| `error.tsx` | Error boundary para o segmento (deve ser 'use client') | Não |
| `not-found.tsx` | UI para `notFound()` ou rota inexistente | Não |
| `route.ts` | API Route Handler (substituiu `pages/api/`) | Não |
| `template.tsx` | Como layout mas re-renderiza em cada navegação | Raro |
| `default.tsx` | Fallback para Parallel Routes | Raro |

## Estrutura de Rotas

```
app/
  layout.tsx           → wraps todos os pages (RootLayout)
  page.tsx             → rota: /

  dashboard/
    layout.tsx         → wraps pages dentro de dashboard/
    page.tsx           → rota: /dashboard

    [id]/
      page.tsx         → rota: /dashboard/123, /dashboard/abc

    [...slug]/
      page.tsx         → rota: /dashboard/a/b/c (catch-all)

    [[...slug]]/
      page.tsx         → rota: /dashboard OU /dashboard/a/b (optional catch-all)
```

## Route Groups — (folder)

Pasta com `()` não contribui para a URL. Útil para layouts diferentes sem mudar a URL.

```
app/
  (auth)/              ← não aparece na URL
    login/
      page.tsx         → rota: /login (não /auth/login)
    signup/
      page.tsx         → rota: /signup

  (protected)/         ← tem layout diferente (com sidebar, header)
    dashboard/
      page.tsx         → rota: /dashboard
    settings/
      page.tsx         → rota: /settings
```

```typescript
// app/(protected)/layout.tsx — layout só para rotas protegidas
export default function ProtectedLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex">
      <Sidebar />
      <main>{children}</main>
    </div>
  )
}
```

## Dynamic Routes

```typescript
// app/viagens/[id]/page.tsx
interface PageProps {
  params: Promise<{ id: string }>        // Next 15: params é Promise
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>
}

export default async function ViagemPage({ params }: PageProps) {
  const { id } = await params           // Next 15: await obrigatório
  const viagem = await getViagem(id)

  return <ViagemDetail viagem={viagem} />
}

// Gerar páginas estáticas (opcional — ISR por defeito)
export async function generateStaticParams() {
  const viagens = await getViagensIds()
  return viagens.map(v => ({ id: v.id }))
}
```

## Metadata API

```typescript
// Static metadata
export const metadata = {
  title: 'Dashboard',
  description: 'Painel de controlo',
}

// Dynamic metadata
export async function generateMetadata({ params }: PageProps) {
  const { id } = await params
  const viagem = await getViagem(id)

  return {
    title: `Viagem ${viagem.numero}`,
    description: viagem.descricao,
  }
}
```

## Parallel Routes e Intercepting Routes

```
app/
  @modal/              ← slot de parallel route
    (.)viagens/[id]/
      page.tsx         ← intercepta /viagens/[id] quando navegado do dashboard
  layout.tsx           ← recebe { children, modal }
  page.tsx
```

```typescript
// app/layout.tsx com parallel route
export default function Layout({
  children,
  modal,
}: {
  children: React.ReactNode
  modal: React.ReactNode  // @modal slot
}) {
  return (
    <>
      {children}
      {modal}
    </>
  )
}
```

## Common Mistakes

```typescript
// ❌ Next 14 pattern — params não é Promise
export default function Page({ params }: { params: { id: string } }) {
  const { id } = params  // funciona em 14, deprecado em 15
}

// ✅ Next 15 — params é Promise
export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
}

// ❌ Múltiplos RootLayouts — cada Route Group com layout.tsx diferente é válido
// MAS só um layout na raiz de app/ pode ter <html><body>
```

## See Also

- [server-components.md](./server-components.md) — RSC vs Client Component
- [../patterns/data-fetching.md](../patterns/data-fetching.md) — buscar dados em page.tsx
- [../patterns/middleware.md](../patterns/middleware.md) — auth guard entre rotas
