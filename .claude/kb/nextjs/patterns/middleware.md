# Middleware no Next.js com Supabase Auth

> **Purpose**: Auth guard, redirect e cookie refresh em middleware.ts
> **MCP Validated**: 2026-06-20

## When to Use

- Proteger rotas autenticadas (redirecionar para login)
- Refresh automático de tokens Supabase em SSR
- Redirects e rewrites baseados em condições de request

## Implementation

### Middleware Supabase Auth (padrão recomendado)

```typescript
// middleware.ts (raiz do projecto)
import { createServerClient } from '@supabase/ssr'
import { NextResponse, type NextRequest } from 'next/server'

export async function middleware(request: NextRequest) {
  let supabaseResponse = NextResponse.next({ request })

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll()
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) =>
            request.cookies.set(name, value)
          )
          supabaseResponse = NextResponse.next({ request })
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options)
          )
        },
      },
    }
  )

  // CRITICAL: getUser() valida o token no servidor — nunca getSession() aqui
  const { data: { user } } = await supabase.auth.getUser()

  // Proteger rotas autenticadas
  if (!user && !request.nextUrl.pathname.startsWith('/login')) {
    const url = request.nextUrl.clone()
    url.pathname = '/login'
    return NextResponse.redirect(url)
  }

  // CRITICAL: retornar supabaseResponse com os cookies actualizados
  return supabaseResponse
}

export const config = {
  matcher: [
    // Excluir assets estáticos e internos do Next.js
    '/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)',
  ],
}
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `matcher` | `'/'` | Padrões de URL onde o middleware corre |
| `runtime` | `'nodejs'` | Edge por defeito — sem Node.js APIs |

## Erros Comuns no Matcher

```typescript
// ❌ Corre em assets estáticos — desnecessário e lento
matcher: ['/:path*']

// ❌ Muito restritivo — não protege sub-rotas
matcher: ['/dashboard']

// ✅ Exclui assets, cobre tudo o resto
matcher: ['/((?!_next/static|_next/image|favicon.ico|api/public).*)',]

// ✅ Para proteger só certas rotas
matcher: ['/dashboard/:path*', '/admin/:path*']
```

## Redirect Condicional por Role

```typescript
export async function middleware(request: NextRequest) {
  // ... setup supabase como acima

  const { data: { user } } = await supabase.auth.getUser()

  if (!user) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  // Verificar role em metadata do user
  const userRole = user.user_metadata?.role

  if (request.nextUrl.pathname.startsWith('/admin') && userRole !== 'admin') {
    return NextResponse.redirect(new URL('/dashboard', request.url))
  }

  return supabaseResponse
}
```

## Example Usage

```typescript
// Estrutura típica de projecto com auth
app/
  (auth)/
    login/page.tsx
    signup/page.tsx
  (protected)/           ← todas as rotas aqui precisam de auth
    dashboard/page.tsx
    viagens/page.tsx

// middleware.ts protege (protected) e redireciona para login
```

## See Also

- [../concepts/server-components.md](../concepts/server-components.md)
- [../../supabase/patterns/server-client.md](../../supabase/patterns/server-client.md)
