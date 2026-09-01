---
name: judge
description: Cross-model second opinion on DEFINE, DESIGN, BUILD_REPORT, or any code file — sends to OpenRouter (non-Claude model) for independent review
---

# Judge Command

> Second opinion layer — envia o documento para um modelo diferente (via OpenRouter) e retorna veredicto PASS/FAIL com evidências. Complementa `/grill-me`, não concorre.

## Como difere de /grill-me

| Comando | Mecanismo | O que testa | Quando usar |
|---------|-----------|-------------|-------------|
| `/grill-me` | Claude faz perguntas adversariais sobre o DEFINE | **Qualidade** — "está spec bom o suficiente?" | Antes de ir para DESIGN |
| `/judge` | Outro modelo (GPT-4o) lê o documento e emite veredicto | **Completude + correctude** — "tem algo errado que Claude não viu?" | Antes de avançar qualquer fase |

São complementares: `/grill-me` melhora qualidade, `/judge` é um segundo par de olhos de outro modelo.

## Usage

```bash
/judge .claude/sdd/features/DEFINE_{FEATURE}.md
/judge .claude/sdd/features/DESIGN_{FEATURE}.md
/judge .claude/sdd/reports/BUILD_REPORT_{FEATURE}.md
/judge src/components/MyComponent.tsx   # qualquer arquivo de código
```

## Setup (uma vez)

```bash
export OPENROUTER_API_KEY=sk-or-v1-...   # openrouter.ai/keys
```

## Processo inline

Quando invocado via Claude Code:

1. Ler o arquivo alvo
2. Auto-detectar fase pelo nome (`DEFINE_*` → phase=define, `DESIGN_*` → phase=design, `BUILD_REPORT_*` → phase=build)
3. Selecionar modelo: define/design → `openai/gpt-4o`, build/generic → `openai/gpt-4o-mini`
4. Enviar via `scripts/judge.py` para OpenRouter
5. Apresentar veredicto estruturado

## Campos validados por fase

### DEFINE (clarity score >= 12/15)
- Problem statement (não vazio, não "TBD")
- User story (persona + ação + benefício)
- Acceptance criteria (mínimo 3, testáveis)
- Out of scope (mínimo 1 explícito)
- Data contracts (input/output)
- Contradições entre seções

### DESIGN (arquitetura soundness)
- APIs/libraries hallusinadas
- Missing edge cases (null, concorrência, idempotência)
- Unsafe defaults (auth permissiva, sem retry, sem timeout)
- Decisões sem justificativa
- Wave scheduling inconsistente
- Security gaps (IAM, RLS, secrets)

### BUILD_REPORT / código (quality >= 90%)
- Logic errors, resource leaks
- SQL injection (f-strings), shell=True, eval()
- Supabase: RLS desabilitado, missing policies
- Next.js: NEXT_PUBLIC_ em secrets, Server Actions sem auth
- IAM: roles/editor em service accounts

## Output

```
## Judge Verdict — PASS

Target: `.claude/sdd/features/DEFINE_ANALYTICS.md`  | Phase: `define` | Model: `openai/gpt-4o` | Confidence: 0.88

Summary: Spec is ready for /design — acceptance criteria are testable and scope is explicit.

_Judge is advisory. Claude is the author; final call is yours._
```

```
## Judge Verdict — FAIL

**Target:** `.claude/sdd/features/DEFINE_ANALYTICS.md` | Phase: `define` | Confidence: 0.62

**Summary:** Spec has gaps that would cause rework at DESIGN phase.

### Concerns

| Severity | Issue | Evidence |
|----------|-------|----------|
| high | Only 1 acceptance criteria — need >= 3 | "Acceptance Criteria" section |
| high | Out of scope not defined | Section missing entirely |
| medium | Data contracts — output schema is "TBD" | "Data Contracts" section |

### Suggested Fixes
- Add 2+ testable ATs (Given/When/Then format)
- Add explicit Out of Scope section with at least 1 item
- Define output schema (even if approximate)
```

## CLI (fora do Claude)

```bash
# Precisa: export OPENROUTER_API_KEY=sk-or-v1-...
python3 scripts/judge.py .claude/sdd/features/DEFINE_FEATURE.md
python3 scripts/judge.py .claude/sdd/features/DESIGN_FEATURE.md --phase design
python3 scripts/judge.py --ledger          # ver uso do dia
python3 scripts/judge.py --json            # output JSON raw
```

## Orçamento

O judge usa um ledger `.claude/storage/judge-ledger.jsonl` para controlar custo.
Padrão: 10 chamadas por dia UTC. Mudar via `export JUDGE_BUDGET=20`.

## Quando usar obrigatoriamente

- Antes de qualquer `/design` em features com implicação de segurança
- Antes de `/ship` em features com mudança de schema
- Como gate adicional ao `/grill-me` em features críticas

## Quando pode pular

- Features pequenas (effort < 2h) onde `/grill-me` já rodou
- Iterações rápidas com `/iterate` em DEFINE já aprovado
