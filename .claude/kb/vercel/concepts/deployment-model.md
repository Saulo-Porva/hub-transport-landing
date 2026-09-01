# Modelo de Deploy no Vercel

> **Purpose**: Como o Vercel constrói e serve aplicações Next.js — runtimes, ambientes, regiões
> **MCP Validated**: 2026-06-20

## Overview

O Vercel analisa a saída do `next build` e decide automaticamente o que é:
- **Static** (HTML/CSS/JS) → servido de CDN global
- **Serverless Function** → Route Handlers, Server Components com dados dinâmicos
- **Edge Function** → Runtime leve, mais rápido, menos APIs disponíveis

## Fluxo de Deploy

```
git push main
    ↓
Vercel detecta Next.js
    ↓
next build (no servidor de build)
    ↓
Análise da saída:
  ├── Rotas estáticas → CDN
  ├── Server Components dinâmicos → Serverless Functions
  ├── Route Handlers → Serverless Functions
  └── Middleware → Edge Functions (sempre)
    ↓
Deploy para produção (swap sem downtime)
```

## Ambientes e Branches

```
main branch          → Production environment
                       URL: https://meu-projecto.vercel.app
                       URL customizado: https://meudominio.com

feature/* branches   → Preview environments
PR #42               → URL: https://meu-projecto-git-feature-abc123.vercel.app

vercel dev (local)   → Development environment
                       URL: http://localhost:3000
```

Cada ambiente tem as **suas próprias variáveis de ambiente** configuradas no dashboard.

## Runtimes

### Node.js (default)

- Acesso a todas as APIs Node.js (fs, crypto, etc.)
- Cold start: ~200-500ms
- Duração: 10s (Hobby), 60s (Pro)
- Ideal para: Route Handlers, Server Actions com BD

### Edge Runtime

```typescript
// Declarar edge runtime por ficheiro
export const runtime = 'edge'

// Disponível: fetch, Web Crypto API, atob/btoa
// NÃO disponível: fs, Buffer, Node.js nativo
```

- Cold start: ~0ms (sempre quente)
- Ideal para: middleware, redirects, auth rápida sem BD
- Restrição: não pode importar a maior parte dos packages Node.js

### Escolher o Runtime

```
Preciso de:
├── Acesso a BD/Supabase → Node.js
├── fs, Buffer, crypto Node.js → Node.js
├── Apenas fetch e lógica simples → Edge
├── Latência muito baixa e global → Edge
└── Middleware Supabase → Edge (default do middleware)
```

## Regiões

- **Default:** iad1 (Washington D.C.) ou a região mais próxima do utilizador
- **Configurável** em vercel.json ou no dashboard
- Functions correm **numa região** (não multi-region sem Pro+)
- **Edge Functions** correm globalmente em todas as regiões Vercel

```json
// vercel.json — definir região da função
{
  "functions": {
    "app/api/**/*.ts": {
      "regions": ["fra1"]  // Frankfurt
    }
  }
}
```

## Preview Deployments

- Criados automaticamente em cada push para branches não-main
- Partilháveis com link único (útil para QA)
- Não afectam produção
- Têm as env vars do ambiente "Preview" (separadas de Production)

```
⚠️ Atenção: Preview usa env vars de Preview, não de Production.
Se uma feature funciona em Preview mas falha em Produção,
verificar se há env vars configuradas só em Production.
```

## Build Cache

O Vercel guarda cache do `node_modules` e da pasta `.next/cache` entre builds.

```bash
# Forçar build limpo (quando cache corrompida)
# Dashboard → Deployments → "..." → Redeploy → desmarcar "Use existing build cache"
```

## See Also

- [env-vars.md](./env-vars.md) — gestão de variáveis de ambiente
- [../patterns/build-config.md](../patterns/build-config.md) — next.config e vercel.json
- [../patterns/debug-functions.md](../patterns/debug-functions.md) — interpretar logs de função
