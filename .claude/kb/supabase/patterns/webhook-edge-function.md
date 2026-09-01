# Webhook Edge Function

> Receber e processar webhooks externos com validação de assinatura e idempotência.

## Problema

Webhooks externos (Stripe, GitHub, WhatsApp) chegam sem autenticação de usuário. Precisam de: validação de assinatura para garantir origem, idempotência para não processar o mesmo evento duas vezes, e resposta rápida (< 5s) para evitar retry do sender.

## Solução

```typescript
// supabase/functions/stripe-webhook/index.ts
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'
import Stripe from 'https://esm.sh/stripe@13'

const stripe = new Stripe(Deno.env.get('STRIPE_SECRET_KEY')!, {
  apiVersion: '2023-10-16',
  httpClient: Stripe.createFetchHttpClient(),
})

const supabase = createClient(
  Deno.env.get('SUPABASE_URL')!,
  Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
)

serve(async (req: Request) => {
  if (req.method !== 'POST') {
    return new Response('Method not allowed', { status: 405 })
  }

  // 1. Ler body como texto (necessário para validação de assinatura)
  const body = await req.text()
  const signature = req.headers.get('stripe-signature')

  // 2. Validar assinatura
  let event: Stripe.Event
  try {
    event = stripe.webhooks.constructEventAsync
      ? await stripe.webhooks.constructEventAsync(
          body,
          signature!,
          Deno.env.get('STRIPE_WEBHOOK_SECRET')!
        )
      : stripe.webhooks.constructEvent(
          body,
          signature!,
          Deno.env.get('STRIPE_WEBHOOK_SECRET')!
        )
  } catch (err) {
    return new Response(`Webhook signature invalid: ${err.message}`, { status: 400 })
  }

  // 3. Idempotência — verificar se já processamos este evento
  const { data: existing } = await supabase
    .from('processed_events')
    .select('id')
    .eq('event_id', event.id)
    .single()

  if (existing) {
    return new Response(JSON.stringify({ received: true, duplicate: true }), {
      headers: { 'Content-Type': 'application/json' },
      status: 200,
    })
  }

  // 4. Processar evento
  try {
    await handleEvent(event)

    // 5. Registrar como processado
    await supabase.from('processed_events').insert({
      event_id: event.id,
      event_type: event.type,
      processed_at: new Date().toISOString(),
    })
  } catch (err) {
    // Logar mas retornar 200 — evitar retry infinito do Stripe
    console.error('Processing error:', err)
    await supabase.from('failed_events').insert({
      event_id: event.id,
      event_type: event.type,
      error: err.message,
    })
  }

  return new Response(JSON.stringify({ received: true }), {
    headers: { 'Content-Type': 'application/json' },
    status: 200,
  })
})

async function handleEvent(event: Stripe.Event) {
  switch (event.type) {
    case 'checkout.session.completed': {
      const session = event.data.object as Stripe.Checkout.Session
      await supabase.from('orders').update({ status: 'paid' })
        .eq('stripe_session_id', session.id)
      break
    }
    case 'customer.subscription.deleted': {
      const sub = event.data.object as Stripe.Subscription
      await supabase.from('subscriptions').update({ status: 'canceled' })
        .eq('stripe_subscription_id', sub.id)
      break
    }
    // Ignorar silenciosamente eventos não tratados
  }
}
```

```sql
-- Tabela de idempotência
CREATE TABLE public.processed_events (
  id           bigserial PRIMARY KEY,
  event_id     text UNIQUE NOT NULL,
  event_type   text NOT NULL,
  processed_at timestamptz DEFAULT now()
);

CREATE TABLE public.failed_events (
  id         bigserial PRIMARY KEY,
  event_id   text NOT NULL,
  event_type text NOT NULL,
  error      text,
  created_at timestamptz DEFAULT now()
);
```

## Variações

**GitHub webhook** — header `x-hub-signature-256`, validação com HMAC-SHA256:
```typescript
const secret = Deno.env.get('GITHUB_WEBHOOK_SECRET')!
const sig = req.headers.get('x-hub-signature-256')?.replace('sha256=', '')
const mac = await crypto.subtle.importKey('raw', new TextEncoder().encode(secret),
  { name: 'HMAC', hash: 'SHA-256' }, false, ['sign'])
const expected = await crypto.subtle.sign('HMAC', mac, new TextEncoder().encode(body))
const expectedHex = Array.from(new Uint8Array(expected))
  .map(b => b.toString(16).padStart(2, '0')).join('')
if (sig !== expectedHex) return new Response('Invalid signature', { status: 401 })
```

**Processamento assíncrono** — retornar 200 imediatamente e enfileirar numa tabela Supabase; worker separado processa em background.
