# Supabase KB — Index

> Entry point for Supabase knowledge: RLS, pgvector/RAG, realtime, edge functions, and SSR auth.

---

## Domain Overview

| Area | Scope | KB File |
|------|-------|---------|
| RLS Policies | Row-level security: roles, policies, debugging empty results | `concepts/rls-policies.md` |
| pgvector | Vector embeddings, similarity search, IVFFlat index, match_documents | `concepts/pgvector-fundamentals.md` |
| Realtime | Subscriptions, broadcast, presence, channel setup | `concepts/realtime.md` |
| Edge Functions | Deno runtime, request handling, CORS, secrets | `concepts/edge-functions.md` |
| Multi-tenant RLS | org_id isolation, team-based access patterns | `patterns/multi-tenant-rls.md` |
| RAG Vector Store | pgvector + embeddings + similarity search pipeline | `patterns/rag-vector-store.md` |
| Webhook Edge Function | Signature validation, idempotency, sync response | `patterns/webhook-edge-function.md` |

---

## Quick Navigation

- **Query returns empty without error?** → `concepts/rls-policies.md` → Debugging section
- **Need to bypass RLS for admin?** → `patterns/multi-tenant-rls.md` → service_role pattern
- **Building RAG?** → `patterns/rag-vector-store.md`
- **Webhook validation?** → `patterns/webhook-edge-function.md`
- **SSR auth session?** → `concepts/rls-policies.md` → SSR section

---

## Client Type Decision

| Client | When | Import |
|--------|------|--------|
| `createServerClient` | Server Components, Server Actions, middleware | `@supabase/ssr` |
| `createBrowserClient` | Client Components only | `@supabase/ssr` |
| `createClient` (service_role) | Admin ops, bypass RLS | `@supabase/supabase-js` + `SUPABASE_SERVICE_ROLE_KEY` |

**Rule:** Never use `SUPABASE_SERVICE_ROLE_KEY` in client-side code. Always `SUPABASE_URL` + `SUPABASE_ANON_KEY` on the browser.

---

## Most Common Errors

| Symptom | Cause | Fix |
|---------|-------|-----|
| Query returns `[]` with no error | RLS blocking anon/authenticated | Add SELECT policy or use service_role |
| `JWT expired` | Session not refreshed in SSR | Use `createServerClient` with cookie adapter |
| `row-level security policy violation` | INSERT/UPDATE without matching policy | Add INSERT/UPDATE policy |
| Realtime not firing | Table not in replication | Enable replication in Supabase Dashboard |
