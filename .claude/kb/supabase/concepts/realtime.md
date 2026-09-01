# Supabase Realtime

> Sistema de subscriptions em tempo real baseado em WebSockets para mudanças de banco, broadcast e presença.

## O que é

Realtime usa uma conexão WebSocket persistente e expõe três canais: **Postgres Changes** (reage a INSERT/UPDATE/DELETE via replication), **Broadcast** (mensagens arbitrárias entre clientes), e **Presence** (rastreia quais usuários estão online em um canal).

## Quando usar

- Atualizar UI automaticamente quando outro usuário modifica dados
- Chat, notificações, live dashboards
- Indicadores de "usuário digitando" ou "quem está vendo"

## Sintaxe / API

```typescript
import { createBrowserClient } from '@supabase/ssr'
const supabase = createBrowserClient(url, anonKey)

// --- Postgres Changes ---
const channel = supabase
  .channel('db-changes')
  .on(
    'postgres_changes',
    {
      event: '*',        // 'INSERT' | 'UPDATE' | 'DELETE' | '*'
      schema: 'public',
      table: 'messages',
      filter: 'room_id=eq.123',   // opcional — filtro server-side
    },
    (payload) => {
      console.log(payload.eventType)  // 'INSERT' | 'UPDATE' | 'DELETE'
      console.log(payload.new)        // novo registro
      console.log(payload.old)        // registro anterior (UPDATE/DELETE)
    }
  )
  .subscribe((status) => {
    // 'SUBSCRIBED' | 'TIMED_OUT' | 'CLOSED' | 'CHANNEL_ERROR'
    console.log(status)
  })

// --- Broadcast (mensagens arbitrárias entre clientes) ---
const broadcast = supabase
  .channel('room:123')
  .on('broadcast', { event: 'cursor-move' }, ({ payload }) => {
    console.log(payload.x, payload.y)
  })
  .subscribe()

// Enviar broadcast
await broadcast.send({
  type: 'broadcast',
  event: 'cursor-move',
  payload: { x: 100, y: 200 },
})

// --- Presence (quem está online) ---
const presence = supabase
  .channel('room:123')
  .on('presence', { event: 'sync' }, () => {
    const state = presence.presenceState()
    console.log(state)  // { userId: [{ online_at, user_id }] }
  })
  .on('presence', { event: 'join' }, ({ key, newPresences }) => {})
  .on('presence', { event: 'leave' }, ({ key, leftPresences }) => {})
  .subscribe(async (status) => {
    if (status === 'SUBSCRIBED') {
      await presence.track({ user_id: 'abc', online_at: new Date().toISOString() })
    }
  })

// --- Cleanup (obrigatório em React useEffect) ---
useEffect(() => {
  const channel = supabase.channel('...').on(...).subscribe()
  return () => { supabase.removeChannel(channel) }
}, [])
```

## Habilitar replicação (Dashboard)

`Database → Replication → Supabase Realtime → Tables` → ativar a tabela.  
Sem isso, Postgres Changes não dispara eventos.

## Armadilhas comuns

- Não remover o canal no cleanup do useEffect — múltiplas subscriptions acumulam e causam memory leak
- Filtros com sintaxe errada (`filter: 'room_id=eq.123'`) — verificar formato `column=operator.value`
- Postgres Changes não dispara para UPDATE/DELETE sem replication identity — adicionar `REPLICA IDENTITY FULL` na tabela
- Broadcast não persiste — mensagens enviadas enquanto cliente desconectado são perdidas
