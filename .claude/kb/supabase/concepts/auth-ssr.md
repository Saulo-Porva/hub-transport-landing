# Auth em Next.js App Router com @supabase/ssr

> **Purpose**: Setup de auth Supabase em contexto SSR — cookies, refresh de token, getUser vs getSession
> **MCP Validated**: 2026-06-20

## Overview

O Supabase usa cookies HttpOnly para persistir sessões em SSR. O `@supabase/ssr` gere o ciclo de vida dos cookies automaticamente, mas o middleware deve estar configurado correctamente para refrescar tokens.

## getUser() vs getSession()

| Método | Valida no servidor? | Seguro para auth? | Quando usar |
|--------|---------------------|-------------------|-------------|
| `auth.getUser()` | ✅ Sim — verifica com Supabase | ✅ Sim | Server Components, middleware, Server Actions |
| `auth.getSession()` | ❌ Não — lê só o cookie local | ⚠️ Não para decisões de auth | Client Components para UI condicional |

```typescript
// ✅ Correcto — valida o token
const { data: { user }, error } = await supabase.auth.getUser()
if (!user) redirect('/login')

// ❌ Inseguro — cookie pode ser manipulado
const { data: { session } } = await supabase.auth.getSession()
if (!session) redirect('/login')  // bypass possível com cookie falsificado
```

## Fluxo de Auth Completo

```
Browser                     Next.js Server              Supabase
   |                              |                          |
   |──POST /auth/login──────────>|                          |
   |                              |──signInWithPassword()──>|
   |                              |<────── JWT token ────────|
   |<── Set-Cookie: sb-* ────────|                          |
   |                              |                          |
   |──GET /dashboard ───────────>|                          |
   |                              |── middleware runs ───────|
   |                              |── getUser() ───────────>|
   |                              |<────── user data ────────|
   |<── page HTML ───────────────|                          |
```

## Login com Email/Password

```typescript
// actions/auth.ts
'use server'
import { createClient } from '@/lib/supabase/server'
import { redirect } from 'next/navigation'

export async function login(formData: FormData) {
  const supabase = await createClient()

  const { error } = await supabase.auth.signInWithPassword({
    email: formData.get('email') as string,
    password: formData.get('password') as string,
  })

  if (error) return { error: error.message }

  redirect('/dashboard')
}

export async function logout() {
  const supabase = await createClient()
  await supabase.auth.signOut()
  redirect('/login')
}
```

## Proteger Server Component

```typescript
// app/dashboard/page.tsx
import { createClient } from '@/lib/supabase/server'
import { redirect } from 'next/navigation'

export default async function DashboardPage() {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()

  if (!user) redirect('/login')  // fora de try/catch

  return <Dashboard userId={user.id} />
}
```

## Ler User em Server Component sem Redirect

```typescript
// Quando a página é pública mas muda com auth
export default async function HomePage() {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()

  return (
    <main>
      {user ? <AuthenticatedNav user={user} /> : <PublicNav />}
      <HeroSection />
    </main>
  )
}
```

## OAuth (Google, GitHub)

```typescript
// actions/auth.ts
export async function signInWithGoogle() {
  const supabase = await createClient()
  const { data, error } = await supabase.auth.signInWithOAuth({
    provider: 'google',
    options: {
      redirectTo: `${process.env.NEXT_PUBLIC_APP_URL}/auth/callback`,
    },
  })

  if (error) return { error: error.message }
  if (data.url) redirect(data.url)
}
```

```typescript
// app/auth/callback/route.ts — troca o code pelo token
import { createClient } from '@/lib/supabase/server'
import { NextResponse } from 'next/server'

export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url)
  const code = searchParams.get('code')

  if (code) {
    const supabase = await createClient()
    await supabase.auth.exchangeCodeForSession(code)
  }

  return NextResponse.redirect(`${origin}/dashboard`)
}
```

## User Metadata e Roles

```typescript
// Aceder ao role do utilizador
const { data: { user } } = await supabase.auth.getUser()
const role = user?.user_metadata?.role  // 'admin' | 'motorista' | undefined

// Actualizar metadata (requer privilégios admin)
const adminClient = createAdminClient()
await adminClient.auth.admin.updateUserById(userId, {
  user_metadata: { role: 'admin' }
})
```

## Problemas Comuns

| Problema | Causa | Fix |
|----------|-------|-----|
| Redirect loop para /login | middleware não a devolver supabaseResponse | Ver middleware.ts — deve retornar `supabaseResponse` não `new NextResponse()` |
| Token não refresca | cookies.setAll não implementado | Implementar setAll no server client |
| `cookies()` não disponível | Await em falta (Next 15) | `const cookieStore = await cookies()` |
| User null em Server Component | Middleware não correu | Verificar matcher config |

## See Also

- [client-types.md](./client-types.md) — setup dos clientes
- [../../nextjs/patterns/middleware.md](../../nextjs/patterns/middleware.md) — middleware auth completo
- [../patterns/server-client.md](../patterns/server-client.md) — setup SSR
