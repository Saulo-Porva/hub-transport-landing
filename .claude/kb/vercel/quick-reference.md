# Vercel Quick Reference

> **Purpose**: Lookup rápido de ambientes, env vars, CLI, erros comuns
> **MCP Validated**: 2026-06-20

## Ambientes

| Ambiente | Quando activo | Branch |
|----------|---------------|--------|
| **Production** | Deploy para branch main/master | `main` |
| **Preview** | Qualquer outro branch ou PR | Todos os outros |
| **Development** | `vercel dev` local | Local |

## Env Vars — Scopes

| Scope | Onde se aplica | Quem vê |
|-------|----------------|---------|
| Production | Deploys de production | Só produção |
| Preview | Deploys de preview | Só previews |
| Development | `vercel dev` | Local com `vercel dev` |
| All | Todos os ambientes | ⚠️ Usar com cuidado para segredos |

## NEXT_PUBLIC_ vs Privadas

| Tipo | Prefixo | Disponível no browser? | Quando usar |
|------|---------|----------------------|-------------|
| Pública | `NEXT_PUBLIC_*` | ✅ Sim (embutida em build) | URLs, chaves anon |
| Privada | Sem prefixo | ❌ Não | Secrets, service roles, API keys |

## CLI Comandos Comuns

```bash
# Login
vercel login

# Deploy preview
vercel

# Deploy production
vercel --prod

# Listar env vars do projecto
vercel env ls

# Adicionar env var
vercel env add NOME_DA_VAR

# Puxar env vars para .env.local
vercel env pull .env.local

# Ver logs em tempo real
vercel logs --follow

# Ver logs de função específica
vercel logs --filter /api/viagens
```

## Erros Comuns de Build

| Erro | Causa | Fix |
|------|-------|-----|
| `Module not found` | devDependency em vez de dependency | Mover para `dependencies` no package.json |
| `Type error` | TypeScript strict em CI | Corrigir tipos (ou `ignoreBuildErrors: true` no next.config como último recurso) |
| `Cannot find module 'X'` | Package não instalado | Verificar package.json e re-fazer deploy |
| `Environment variable X not defined` | Var não configurada no ambiente | Adicionar no dashboard Vercel |
| `Exceeded maximum lambda size` | Bundle demasiado grande | Analisar com `@next/bundle-analyzer` |

## Erros Comuns de Runtime (Function Logs)

| Log | Causa | Fix |
|-----|-------|-----|
| `Function timeout` | Execução > 10s (hobby) / 60s (pro) | Optimizar query ou usar streaming |
| `Memory: 128 MB Used: 200 MB` | Memory exceeded | Aumentar limite em vercel.json |
| `FUNCTION_INVOCATION_FAILED` | Crash inesperado | Ver stack trace nos logs |
| `500: INTERNAL_SERVER_ERROR` | Exception não tratada em Route Handler | Adicionar try/catch e retornar error response |

## Limites por Plano

| Recurso | Hobby | Pro |
|---------|-------|-----|
| Function duration | 10s | 60s (300s com streaming) |
| Function memory | 1024 MB | 3008 MB |
| Payload size | 4.5 MB | 4.5 MB |
| Regions | 1 | Multi-region |

## Ver Logs no Dashboard

1. Vercel Dashboard → Projecto
2. "Deployments" → seleccionar deployment
3. "Functions" tab → seleccionar função
4. "Logs" → filtrar por timestamp ou status

## See Also

- [concepts/env-vars.md](concepts/env-vars.md)
- [patterns/debug-functions.md](patterns/debug-functions.md)
- [patterns/build-config.md](patterns/build-config.md)
