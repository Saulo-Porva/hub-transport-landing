# Caching no Next.js App Router

> **Purpose**: Entender e controlar o comportamento de cache — evitar dados desactualizados
> **Confidence**: 0.95

## Overview

O App Router tem 4 camadas de cache. A causa mais comum de "dados não actualizam" é não chamar `revalidatePath`/`revalidateTag` após mutações, ou confiar no comportamento de cache em desenvolvimento (que é diferente de produção).

## As 4 Camadas de Cache

| Layer | O que cacheia | Duração padrão | Invalidar com |
|-------|---------------|----------------|---------------|
| **Request Memoization** | Resultados de `fetch` idênticos no mesmo request | Um request | Automático |
| **Data Cache** | Resultados de `fetch` entre requests | Indefinido | `revalidateTag`, `revalidatePath`, `no-store` |
| **Full Route Cache** | HTML + RSC payload da rota | Até invalidação | `revalidatePath` |
| **Router Cache** | Navegações recentes no browser | 30s (pages) / 5min (layouts) | `router.refresh()` |

## Controlar o Cache

### Por fetch

```typescript
// Sem cache — sempre busca dados frescos
const data = await fetch('/api/data', { cache: 'no-store' })

// Cache com revalidação por tempo (ISR)
const data = await fetch('/api/data', { next: { revalidate: 60 } })

// Cache com tag — invalidar sob demanda
const data = await fetch('/api/data', { next: { tags: ['viagens'] } })
```

### Por rota (segment config)

```typescript
// page.tsx ou layout.tsx
export const dynamic = 'force-dynamic'   // sem cache para toda a rota
export const revalidate = 60             // ISR — revalida a cada 60s
export const fetchCache = 'force-no-store'  // sem cache em todos os fetches da rota
```

### Após mutação (Server Action)

```typescript
'use server'
import { revalidatePath, revalidateTag } from 'next/cache'

export async function updateViagem(id: string, data: Partial<Viagem>) {
  await db.update(id, data)

  revalidatePath('/viagens')              // invalida a lista
  revalidatePath(`/viagens/${id}`)        // invalida o detalhe
  revalidateTag('viagens')               // invalida todos os fetches com tag 'viagens'
}
```

## Comportamento em Desenvolvimento vs Produção

```
Desenvolvimento (next dev):
  - Full Route Cache: DESACTIVADO
  - Data Cache: ACTIVO mas com menor TTL
  - Requests são sempre frescos por defeito

Produção (next build + next start):
  - Todos os caches ACTIVOS
  - Página estática por defeito se não há dados dinâmicos
  - TESTAR SEMPRE com next build para confirmar comportamento
```

**Erro comum:** Feature funciona em `next dev` mas dados não actualizam em produção — esquecimento de `revalidatePath` que não faz falta em dev.

## Quando Usar Cada Estratégia

| Caso | Estratégia |
|------|------------|
| Dados de configuração raramente mudam | `revalidate: 3600` (1h) |
| Lista de itens que muda com acções do user | `revalidatePath` após mutação |
| Dados em tempo real | `cache: 'no-store'` + `dynamic = 'force-dynamic'` |
| Página de produto/conteúdo | `revalidateTag` com ISR |
| Dashboard de admin com dados sensíveis | `cache: 'no-store'` |

## Common Mistakes

### Wrong — Assumir que dev = prod

```typescript
// Em dev parece funcionar sem revalidate
// Em prod os dados ficam desactualizados
export async function criarViagem(data) {
  await db.insert(data)
  // ❌ sem revalidatePath — em prod a lista não actualiza
}

// ✅
export async function criarViagem(data) {
  await db.insert(data)
  revalidatePath('/viagens')
}
```

### Wrong — revalidatePath com path errado

```typescript
// ❌ path sem /leading-slash não funciona
revalidatePath('viagens')

// ❌ path com parâmetros não resolvidos
revalidatePath('/viagens/[id]')  // não invalida /viagens/123

// ✅
revalidatePath('/viagens')              // invalida a lista
revalidatePath(`/viagens/${id}`)        // invalida o item específico
revalidatePath('/viagens', 'layout')    // invalida layout e pages
```

## Related

- [server-actions.md](./server-actions.md) — onde chamar revalidatePath
- [../patterns/data-fetching.md](../patterns/data-fetching.md) — fetch patterns com cache tags
