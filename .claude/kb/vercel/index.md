# Vercel Knowledge Base

> **Purpose**: Deploy Next.js no Vercel — variáveis de ambiente, function logs, build config, debugging
> **MCP Validated**: 2026-06-20

## Quick Navigation

### Concepts

| File | Purpose |
|------|---------|
| [concepts/deployment-model.md](concepts/deployment-model.md) | Como o Vercel deploy funciona — ambientes, builds, edge vs node |
| [concepts/env-vars.md](concepts/env-vars.md) | Scopes de env vars, NEXT_PUBLIC_, secretos e boas práticas |

### Patterns

| File | Purpose |
|------|---------|
| [patterns/debug-functions.md](patterns/debug-functions.md) | Ler Function Logs, interpretar erros, diagnosticar crashes |
| [patterns/build-config.md](patterns/build-config.md) | next.config, vercel.json, headers, rewrites, output modes |

---

## Quick Reference

- [quick-reference.md](quick-reference.md) — lookup rápido de ambientes, erros, comandos CLI

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Environments** | Production / Preview / Development — cada um com env vars separadas |
| **Serverless Functions** | Ficheiros em `app/api/` e Server Actions — default runtime Node.js |
| **Edge Runtime** | Runtime mais rápido, menos APIs — declarado com `export const runtime = 'edge'` |
| **Function Logs** | Logs de Serverless Functions visíveis no dashboard Vercel |
| **NEXT_PUBLIC_** | Vars com este prefixo são embutidas no bundle do browser em build time |

---

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| `vercel-specialist` | todos | Deploy, env vars, logs, build errors |
| `nextjs-specialist` | patterns/build-config.md | Configuração next.config |
