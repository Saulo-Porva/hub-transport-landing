# Edge Functions

> Funções serverless Deno executadas no edge da Supabase — ideal para webhooks, lógica server-side e integrações com APIs externas.

## O que é

Edge Functions são funções Deno (TypeScript nativo) deployadas na infraestrutura Supabase. Executam próximo ao usuário, têm acesso às variáveis de ambiente do projeto, e são chamadas via HTTP. Não têm acesso direto ao banco — usam o SDK com service_role para operações admin.

## Quando usar

- Receber webhooks externos (Stripe, GitHub, WhatsApp)
- Lógica que não pode estar no cliente (segredos, validações críticas)
- Transformações antes de inserir no banco
- Enviar emails/notificações após um evento

## Sintaxe / API

```typescript
// supabase/functions/process-webhook/index.ts
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
}

serve(async (req: Request) => {
  // Preflight CORS
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders })
  }

  try {
    const body = await req.json()

    // Acesso ao banco com service_role (bypassa RLS)
    const supabase = createClient(
      Deno.env.get('SUPABASE_URL')!,
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
    )

    const { error } = await supabase
      .from('events')
      .insert({ payload: body, received_at: new Date().toISOString() })

    if (error) throw error

    return new Response(
      JSON.stringify({ success: true }),
      { headers: { ...corsHeaders, 'Content-Type': 'application/json' }, status: 200 }
    )
  } catch (err) {
    return new Response(
      JSON.stringify({ error: err.message }),
      { headers: { ...corsHeaders, 'Content-Type': 'application/json' }, status: 400 }
    )
  }
})
```

```bash
# Deploy
supabase functions deploy process-webhook

# Teste local (requer Docker)
supabase functions serve process-webhook --env-file supabase/.env.local

# Invocar via CLI
supabase functions invoke process-webhook --body '{"test": true}'

# Secrets (não usar .env em produção — usar secrets do projeto)
supabase secrets set MY_API_KEY=abc123
supabase secrets list
```

## Autenticação na Edge Function

```typescript
// Verificar JWT do usuário chamador
const authHeader = req.headers.get('Authorization')
const token = authHeader?.replace('Bearer ', '')

const supabase = createClient(url, anonKey, {
  global: { headers: { Authorization: `Bearer ${token}` } }
})

const { data: { user }, error } = await supabase.auth.getUser()
if (error || !user) {
  return new Response('Unauthorized', { status: 401 })
}
```

## Armadilhas comuns

- Usar `Deno.env.get` sem `!` — retorna `undefined` silenciosamente quando secret não configurado
- Esquecer o handler OPTIONS para CORS preflight — browser bloqueia chamadas cross-origin
- Committar `SUPABASE_SERVICE_ROLE_KEY` em `.env` — usar `supabase secrets set` para produção
- Import de npm sem `https://esm.sh/` — Deno não resolve node_modules
- Timeout padrão de 150s — operações longas precisam de workaround via Pub/Sub ou background job
