# Supabase Quick Reference

## RLS Cheatsheet

```sql
-- Enable RLS (mandatory before creating policies)
ALTER TABLE public.orders ENABLE ROW LEVEL SECURITY;

-- SELECT: authenticated users see own rows
CREATE POLICY "users_select_own" ON public.orders
  FOR SELECT USING (auth.uid() = user_id);

-- INSERT: authenticated users insert own rows
CREATE POLICY "users_insert_own" ON public.orders
  FOR INSERT WITH CHECK (auth.uid() = user_id);

-- UPDATE: authenticated users update own rows
CREATE POLICY "users_update_own" ON public.orders
  FOR UPDATE USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);

-- Admin bypass (service_role skips RLS automatically)
-- No policy needed when using service_role key

-- Public read (anon)
CREATE POLICY "public_read" ON public.products
  FOR SELECT USING (true);

-- Check current user role in a policy
auth.role()     -- 'anon' | 'authenticated' | 'service_role'
auth.uid()      -- uuid of authenticated user, null if anon
auth.jwt()      -- full JWT payload as jsonb
```

## Client Types

```typescript
// Server-side (Server Component / Server Action / middleware)
import { createServerClient } from '@supabase/ssr'
const supabase = createServerClient(url, anonKey, { cookies })

// Client-side (Client Component)
import { createBrowserClient } from '@supabase/ssr'
const supabase = createBrowserClient(url, anonKey)

// Admin (bypasses RLS — server only, never expose key)
import { createClient } from '@supabase/supabase-js'
const admin = createClient(url, serviceRoleKey)
```

## pgvector Cheatsheet

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Column type
embedding vector(1536)  -- OpenAI ada-002
embedding vector(768)   -- text-embedding-3-small

-- Index (create after bulk insert)
CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);

-- Similarity search
SELECT id, content, 1 - (embedding <=> query_embedding) AS similarity
FROM documents
ORDER BY embedding <=> query_embedding
LIMIT 5;
```

## Realtime Cheatsheet

```typescript
const channel = supabase
  .channel('room:123')
  .on('postgres_changes', { event: '*', schema: 'public', table: 'messages' },
    (payload) => console.log(payload))
  .subscribe()

// Cleanup
supabase.removeChannel(channel)
```

## Edge Function Cheatsheet

```typescript
// supabase/functions/my-fn/index.ts
import { serve } from 'https://deno.land/std/http/server.ts'
serve(async (req) => {
  const { name } = await req.json()
  return new Response(JSON.stringify({ hello: name }), {
    headers: { 'Content-Type': 'application/json' },
  })
})
```

```bash
supabase functions deploy my-fn
supabase functions serve my-fn --env-file .env.local
```

## Debugging Empty Results

```sql
-- Check if RLS is enabled
SELECT tablename, rowsecurity FROM pg_tables WHERE schemaname = 'public';

-- Check existing policies
SELECT * FROM pg_policies WHERE tablename = 'orders';

-- Test as anon user
SET role anon;
SELECT * FROM orders;
RESET role;

-- Test as authenticated user
SET role authenticated;
SET request.jwt.claim.sub = 'user-uuid-here';
SELECT * FROM orders;
RESET role;
```
