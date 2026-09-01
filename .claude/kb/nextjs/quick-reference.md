# Next.js Quick Reference

## Directives

| Directive | Onde | Efeito |
|-----------|------|--------|
| `'use client'` | Topo do ficheiro | Torna o módulo Client Component — pode usar hooks, events, browser APIs |
| `'use server'` | Topo do ficheiro ou função | Marca como Server Action — roda no servidor, chamável do client |

## File Conventions (App Router)

| Ficheiro | Propósito |
|----------|-----------|
| `page.tsx` | UI da rota — único ficheiro que torna uma rota acessível |
| `layout.tsx` | Wrapper persistente — não re-renderiza entre navegações |
| `loading.tsx` | Skeleton/spinner automático com Suspense |
| `error.tsx` | Error boundary para a rota — deve ter `'use client'` |
| `not-found.tsx` | UI para 404 na rota |
| `route.ts` | Route Handler (endpoint HTTP) — não pode coexistir com page.tsx |
| `middleware.ts` | Corre antes de cada request — raiz do projecto |
| `global-error.tsx` | Error boundary do layout raiz — deve ter `'use client'` |

## Cache Options (fetch)

| Option | Comportamento |
|--------|---------------|
| (padrão) | Cacheia indefinidamente (como `force-cache`) |
| `cache: 'no-store'` | Sem cache — sempre busca dados frescos |
| `cache: 'force-cache'` | Força cache mesmo em rotas dinâmicas |
| `next: { revalidate: 60 }` | ISR — revalida a cada 60 segundos |
| `next: { tags: ['posts'] }` | Cache com tag — invalida com revalidateTag |

## Revalidação Após Mutação

```typescript
import { revalidatePath, revalidateTag } from 'next/cache'

// Invalidar rota específica
revalidatePath('/dashboard')
revalidatePath('/viagens/[id]', 'page')  // só pages, não layouts

// Invalidar por tag
revalidateTag('viagens')

// Redirect após Server Action (comum)
import { redirect } from 'next/navigation'
redirect('/dashboard')
```

## Server Action — Estrutura Mínima

```typescript
// actions/viagem.ts
'use server'

import { revalidatePath } from 'next/cache'

export async function criarViagem(formData: FormData) {
  const destino = formData.get('destino') as string
  // ... lógica
  revalidatePath('/viagens')
}
```

## Variáveis de Ambiente

| Tipo | Acessível em | Exemplo |
|------|-------------|---------|
| `NEXT_PUBLIC_*` | Client + Server | `NEXT_PUBLIC_SUPABASE_URL` |
| Sem prefixo | Server only | `SUPABASE_SERVICE_ROLE_KEY` |

## Dynamic Routes

| Segmento | Exemplo | Match |
|----------|---------|-------|
| `[id]` | `app/viagens/[id]/page.tsx` | `/viagens/123` |
| `[...slug]` | `app/docs/[...slug]/page.tsx` | `/docs/a/b/c` |
| `[[...slug]]` | opcional | `/docs` e `/docs/a/b` |
| `(grupo)` | `app/(dashboard)/page.tsx` | Sem efeito na URL |

## Erros Comuns

| Erro | Causa | Fix |
|------|-------|-----|
| "Hydration mismatch" | Diferença server/client render | `Date`, `Math.random()`, ou window em RSC |
| "async is not allowed in Client" | async em Client Component | Mover para Server Component |
| "cookies() was called outside" | cookies() em contexto não-request | Passar cookies de page/layout |
| "useRouter outside Router" | Hook em Server Component | Adicionar 'use client' |
| "NEXT_REDIRECT" em try/catch | redirect() lança exceção | Chamar redirect() fora do try |
