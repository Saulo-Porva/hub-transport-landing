# Debugging Serverless Functions no Vercel

> **Purpose**: Ler Function Logs, interpretar erros, diagnosticar crashes e timeouts
> **MCP Validated**: 2026-06-20

## Onde Ver os Logs

### Dashboard Vercel

```
Dashboard → Projecto → Deployments → [deployment específico]
  → "Functions" tab
  → Seleccionar função (ex: /api/viagens)
  → "Logs" tab
```

### CLI em Tempo Real

```bash
# Todos os logs do deployment mais recente
vercel logs --follow

# Filtrar por rota
vercel logs --filter /api/viagens --follow

# Logs dos últimos N minutos
vercel logs --since 30m

# Logs de deployment específico
vercel logs dpl_xxxx
```

## Anatomia de um Log de Função

```
2026-06-20T10:23:14.123Z  REQUEST  GET /api/viagens/123
2026-06-20T10:23:14.456Z  LOG      User: abc-123
2026-06-20T10:23:14.678Z  LOG      Fetching from DB...
2026-06-20T10:23:15.100Z  RESPONSE 200  Duration: 977ms  Memory: 84 MB / 1024 MB
```

| Campo | Significado |
|-------|-------------|
| Timestamp | UTC — converter para fuso local |
| `REQUEST` | Início do request |
| `LOG` | `console.log()` do código |
| `ERROR` | `console.error()` ou exception |
| `RESPONSE` | Fim — status code, duração, memória usada |
| Duration | Tempo total de execução |
| Memory | Pico de uso / limite |

## Erros Comuns e Diagnóstico

### FUNCTION_INVOCATION_TIMEOUT

```
FUNCTION_INVOCATION_TIMEOUT  Duration: 10001ms

Causa: Função excedeu o limite de 10s (Hobby) ou 60s (Pro)

Diagnóstico:
1. Identificar a operação lenta nos logs (última LOG antes do timeout)
2. Verificar queries sem índice (SELECT sem WHERE em tabelas grandes)
3. Verificar chamadas a APIs externas lentas

Fix:
- Adicionar índices no Supabase
- Optimizar query (limitar rows, select só campos necessários)
- Usar streaming para operações longas
- Upgrade para Pro se operação é genuinamente longa
```

### FUNCTION_INVOCATION_FAILED

```
FUNCTION_INVOCATION_FAILED  Status: 500

Causa: Crash/exception não tratada

Diagnóstico: Ver a linha de ERROR nos logs antes do RESPONSE
  2026-06-20T10:23:14.678Z  ERROR  TypeError: Cannot read properties of undefined
  at getViagem (/var/task/app/api/viagens/[id]/route.ts:23:15)

Fix: Adicionar try/catch e retornar Response com status 500
```

```typescript
// Route Handler com erro tratado
export async function GET(
  request: Request,
  { params }: { params: Promise<{ id: string }> }
) {
  try {
    const { id } = await params
    const supabase = await createClient()
    const { data, error } = await supabase.from('viagens').select('*').eq('id', id).single()

    if (error) return Response.json({ error: error.message }, { status: 404 })

    return Response.json(data)
  } catch (err) {
    console.error('Unexpected error:', err)
    return Response.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

### Memory Exceeded

```
Memory: 1024 MB / 1024 MB  → próximo do limite
Memory: 1100 MB / 1024 MB  → excedeu (função termina)

Diagnóstico:
- Carregar ficheiros grandes em memória (ex: exportação CSV de 50k rows)
- Dependências pesadas no bundle

Fix:
- Streaming para exportações grandes (não carregar tudo em memória)
- Aumentar limite em vercel.json:
```

```json
{
  "functions": {
    "app/api/export/*.ts": {
      "memory": 3008
    }
  }
}
```

### 401/403 — Auth Issues

```
RESPONSE 401  → Authorization header em falta ou inválido
RESPONSE 403  → Token válido mas sem permissão

Diagnóstico nos logs:
  LOG  User: null   ← getUser() retornou null

Fix: Verificar se cookie está a ser passado correctamente
     (middleware precisa de estar configurado)
```

## Logging Boas Práticas

```typescript
// ✅ Structured logging — fácil de filtrar nos logs Vercel
console.log(JSON.stringify({
  event: 'viagem_created',
  userId: user.id,
  viagemId: data.id,
  duration_ms: Date.now() - startTime,
}))

// ✅ Log de errors com contexto
console.error(JSON.stringify({
  event: 'db_error',
  operation: 'insert_viagem',
  error: error.message,
  userId: user?.id,
}))

// ❌ PII nos logs — Vercel retém logs e podem ser vistos
console.log(`User email: ${user.email}`)   // evitar
console.log(`Password: ${password}`)        // NUNCA
```

## Diagnóstico Remoto Sem Acesso a Prod

```typescript
// Adicionar correlation ID para rastrear requests
export async function GET(request: Request) {
  const requestId = crypto.randomUUID()
  console.log(JSON.stringify({ event: 'request_start', requestId, url: request.url }))

  try {
    // ...
    console.log(JSON.stringify({ event: 'request_success', requestId }))
    return Response.json(data)
  } catch (err) {
    console.error(JSON.stringify({ event: 'request_error', requestId, error: String(err) }))
    return Response.json({ error: 'Internal server error', requestId }, { status: 500 })
  }
}
```

## See Also

- [../concepts/deployment-model.md](../concepts/deployment-model.md) — runtimes e limites
- [../concepts/env-vars.md](../concepts/env-vars.md) — vars indefinidas que causam crashes
- [../../supabase/patterns/debug-rls.md](../../supabase/patterns/debug-rls.md) — erros de BD em functions
