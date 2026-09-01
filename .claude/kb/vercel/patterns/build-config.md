# Build Config — next.config e vercel.json

> **Purpose**: Configurar Next.js e Vercel para deploy — headers, rewrites, output, function config
> **MCP Validated**: 2026-06-20

## next.config.ts — Configurações Chave

```typescript
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  // Imagens de domínios externos
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'xxxxx.supabase.co',
        pathname: '/storage/v1/object/public/**',
      },
    ],
  },

  // Reescrever URLs (sem redirect — URL não muda no browser)
  async rewrites() {
    return [
      {
        source: '/api/legacy/:path*',
        destination: 'https://api-antiga.exemplo.com/:path*',
      },
    ]
  },

  // Headers HTTP em respostas
  async headers() {
    return [
      {
        source: '/api/:path*',
        headers: [
          { key: 'Access-Control-Allow-Origin', value: '*' },
          { key: 'Access-Control-Allow-Methods', value: 'GET,POST,PUT,DELETE' },
        ],
      },
    ]
  },

  // Redirects permanentes e temporários
  async redirects() {
    return [
      {
        source: '/viagens-antigas',
        destination: '/viagens',
        permanent: true,  // 308 — SEO friendly
      },
    ]
  },
}

export default nextConfig
```

## Output Modes

```typescript
// Padrão — sem configuração de output
// Next.js decide: static, SSR, ou serverless — automaticamente

// output: 'standalone' — bundle autossuficiente para Docker/containers
const nextConfig: NextConfig = {
  output: 'standalone',
  // Inclui node_modules necessários no .next/standalone
  // Ideal para deploy em Cloud Run, não necessário no Vercel
}

// output: 'export' — export estático puro (sem Server Components dinâmicos)
const nextConfig: NextConfig = {
  output: 'export',
  // Gera ficheiros HTML estáticos em /out
  // SEM Server Components dinâmicos, Server Actions, ou Route Handlers
  // NÃO usar se tiver Supabase auth ou dados dinâmicos
}
```

## vercel.json — Configuração Vercel-específica

```json
{
  "functions": {
    "app/api/export/*.ts": {
      "memory": 3008,
      "maxDuration": 60
    },
    "app/api/webhooks/*.ts": {
      "maxDuration": 30
    }
  },
  "regions": ["fra1"],
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "X-Content-Type-Options",
          "value": "nosniff"
        },
        {
          "key": "X-Frame-Options",
          "value": "DENY"
        }
      ]
    }
  ]
}
```

## TypeScript Strict em CI

O Vercel corre `next build` com TypeScript. Erros de tipo bloqueiam o deploy.

```typescript
// next.config.ts — opções para CI
const nextConfig: NextConfig = {
  typescript: {
    // ⚠️ Último recurso — ignora erros TS no build
    // Resolver os tipos é sempre preferível
    ignoreBuildErrors: false,  // default: false
  },
  eslint: {
    // ⚠️ Ignorar ESLint no build — se houver erros de lint
    ignoreDuringBuilds: false,  // default: false
  },
}
```

**Preferir sempre** corrigir os erros de tipo a usar `ignoreBuildErrors`.

## Variáveis de Ambiente em next.config

```typescript
const nextConfig: NextConfig = {
  // Validar vars obrigatórias em build time
  // (falha o build se não estiverem definidas)
  env: {
    CUSTOM_BUILD_VAR: process.env.CUSTOM_BUILD_VAR ?? (() => {
      throw new Error('CUSTOM_BUILD_VAR must be defined')
    })(),
  },
}
```

## Bundle Analyzer

```bash
# Instalar
npm install @next/bundle-analyzer

# next.config.ts
import bundleAnalyzer from '@next/bundle-analyzer'

const withBundleAnalyzer = bundleAnalyzer({
  enabled: process.env.ANALYZE === 'true',
})

export default withBundleAnalyzer(nextConfig)

# Correr análise
ANALYZE=true next build
```

## Problemas Comuns de Build

```
Erro: Module not found: Can't resolve 'fs'
Causa: Package com dependência Node.js a ser importado em Client Component
Fix: Mover import para Server Component, ou usar alias:

// next.config.ts
webpack: (config, { isServer }) => {
  if (!isServer) {
    config.resolve.fallback = { fs: false, path: false }
  }
  return config
}

---

Erro: Large Page Data warning
Causa: Dados passados do Server Component para Client Component via props são grandes
Fix: Passar só os campos necessários, não objectos completos

---

Erro: Dynamic server usage — export const dynamic must be set
Causa: Usar cookies(), headers(), searchParams em rota marcada como static
Fix: Adicionar export const dynamic = 'force-dynamic' na page/layout
```

## See Also

- [../concepts/deployment-model.md](../concepts/deployment-model.md) — runtimes e regiões
- [../concepts/env-vars.md](../concepts/env-vars.md) — variáveis em build
- [../../nextjs/concepts/caching.md](../../nextjs/concepts/caching.md) — cache e revalidação
