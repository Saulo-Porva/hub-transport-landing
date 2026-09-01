# Server Components vs Client Components

> **Purpose**: Entender a fronteira RSC/Client — evitar anti-patterns e erros de hidratação
> **Confidence**: 0.97

## Overview

Por defeito no App Router, todos os componentes são Server Components (RSC). Só tornam-se Client Components com `'use client'` no topo do ficheiro. A escolha errada causa: hydration errors, bundle desnecessário no client, ou funcionalidades quebradas.

## Quando Usar Cada Um

| Preciso de... | Usar |
|---------------|------|
| Fetch de dados, async/await | Server Component |
| Acesso a `process.env` (privado) | Server Component |
| Acesso a `cookies()`, `headers()` | Server Component |
| `useState`, `useReducer` | Client Component |
| `useEffect`, `useRef` | Client Component |
| Event listeners (onClick, onChange) | Client Component |
| `window`, `localStorage`, `document` | Client Component |
| Bibliotecas de terceiros com hooks | Client Component |

## A Fronteira (Boundary)

```
page.tsx (Server)
  └── Layout.tsx (Server)
        └── Sidebar.tsx (Server)           ← pode ser async
              └── SearchInput.tsx          ← 'use client' aqui
                    └── Dropdown.tsx       ← também vira client (contaminação)
```

**Regra:** `'use client'` contamina toda a subárvore. Colocar o mais abaixo possível.

## Patterns

### Correcto — Isolar Client Component

```typescript
// PostList.tsx (Server) — fetch de dados
import { LikeButton } from './LikeButton'

export async function PostList() {
  const posts = await fetchPosts()  // async ok em Server Component

  return posts.map(post => (
    <div key={post.id}>
      <h2>{post.title}</h2>
      <LikeButton postId={post.id} />  {/* Client Component só onde necessário */}
    </div>
  ))
}

// LikeButton.tsx (Client) — só o que precisa de interactividade
'use client'
import { useState } from 'react'

export function LikeButton({ postId }: { postId: string }) {
  const [liked, setLiked] = useState(false)
  return <button onClick={() => setLiked(!liked)}>...</button>
}
```

### Errado — 'use client' no layout

```typescript
// ❌ layout.tsx
'use client'  // torna TUDO neste layout client — perde Server Components
import { usePathname } from 'next/navigation'

export default function Layout({ children }) {
  const pathname = usePathname()
  // ...
}

// ✅ Solução — componente separado para o hook
// NavWrapper.tsx
'use client'
import { usePathname } from 'next/navigation'
export function NavWrapper({ children }) { ... }

// layout.tsx — continua Server Component
import { NavWrapper } from './NavWrapper'
export default function Layout({ children }) {
  return <NavWrapper>{children}</NavWrapper>
}
```

## Hydration Mismatch — Diagnóstico

Ocorre quando o HTML gerado no servidor difere do que React tenta hidratar no cliente.

### Causas comuns:

```typescript
// ❌ Date no server vs client pode ser diferente
export default function Page() {
  return <p>Hoje: {new Date().toLocaleDateString()}</p>
}

// ✅ Solução: usar suppressHydrationWarning ou Client Component
'use client'
import { useEffect, useState } from 'react'
export function DataActual() {
  const [data, setData] = useState('')
  useEffect(() => { setData(new Date().toLocaleDateString()) }, [])
  return <p>Hoje: {data}</p>
}

// ❌ Math.random() diferente em cada render
const id = Math.random()  // nunca em componente

// ❌ window/localStorage directo em render
const theme = localStorage.getItem('theme')  // crash no server
```

## Common Mistakes

### Wrong — async em Client Component

```typescript
'use client'
// ❌ Client Components não podem ser async
export default async function MyComponent() {
  const data = await fetchData()  // não funciona
  return <div>{data}</div>
}
```

### Correct — fetch no Server, interactividade no Client

```typescript
// DataPage.tsx (Server)
export default async function DataPage() {
  const data = await fetchData()
  return <InteractiveList items={data} />  // passa dados como props
}

// InteractiveList.tsx
'use client'
export function InteractiveList({ items }) {
  const [filter, setFilter] = useState('')
  // ...
}
```

## Related

- [app-router.md](./app-router.md) — file conventions e layouts
- [../patterns/data-fetching.md](../patterns/data-fetching.md) — fetch patterns em Server Components
