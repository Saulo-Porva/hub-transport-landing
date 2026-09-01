# SDD — Spec-Driven Development

> Build what you spec. Ship what you build. Learn from what you shipped.

---

## O que é SDD?

Implementação sem especificação é a principal causa de retrabalho em projetos de software. Quando o desenvolvedor começa a codar sem um contrato claro — requisitos validados, arquitetura decidida, critérios de aceitação mensuráveis — a tendência é descobrir os problemas no meio da build, ou pior, depois do ship. O custo de corrigir uma decisão arquitetural na fase 3 é 10× o custo de corrigi-la na fase 1.

SDD (Spec-Driven Development) resolve isso com 5 fases sequenciais, cada uma produzindo um artefato versionado em git. Cada artefato serve como contrato entre fases: o DEFINE é o contrato do problema, o DESIGN é o contrato da solução, o BUILD_REPORT é o contrato de verificação. Nenhuma fase começa sem o contrato da fase anterior aprovado.

O diferencial do SDD deste template são três mecanismos ausentes em outros workflows: **SEALED envelope testing** (holdout set que garante que a implementação não foi "ajustada" para passar nos testes visíveis), **quality gate numérico** (score ≥ 90% é pré-requisito para ship), e **security gate integrado** (4 agentes de segurança bloqueiam o ship se encontrarem CRITICAL findings).

---

## As 5 Fases

| Fase | Comando | Input | Output | Quando usar | Agente |
|------|---------|-------|--------|-------------|--------|
| 0 | `/brainstorm` | Ideia vaga | `BRAINSTORM_{FEATURE}.md` | Ideia ainda exploratória | brainstorm-agent |
| 1 | `/define` | BRAINSTORM ou notas | `DEFINE_{FEATURE}.md` + `SEALED_{FEATURE}.md` | Requisitos prontos para captura | define-agent |
| 2 | `/design` | DEFINE aprovado | `DESIGN_{FEATURE}.md` | Feature complexa (ver critérios) | design-agent |
| 3 | `/build` | DESIGN aprovado | Código + `BUILD_REPORT_{FEATURE}.md` | Implementação | build-agent |
| 4 | `/ship` | Build completo | `archive/{FEATURE}/SHIPPED_{DATE}.md` | Pronto para arquivar | ship-agent |

---

### Fase 0: Brainstorm (`/brainstorm`)

**O que produz:** Documento de exploração com abordagens avaliadas, trade-offs, e direção recomendada.

**Critério de entrada:** Qualquer ideia — pode ser vaga. Quanto mais incerta, mais útil o brainstorm.

**Critério de saída:** Abordagem escolhida com justificativa. Usuário aprova a direção.

**Artefato:** `.claude/sdd/features/BRAINSTORM_{FEATURE}.md`

**Quando pular:** Feature já bem definida — vá direto para `/define`.

**stop_condition:** Se já existem ≥ 3 features ativas, o agente recusa abrir nova até que uma seja shippada.

---

### Fase 1: Define (`/define`)

**O que produz:** DEFINE doc com problema, user story, critérios de aceitação, out-of-scope, e contratos de dados. Também cria o arquivo SEALED com cenários de teste que o build-agent nunca verá.

**Critério de entrada:** Ideia suficientemente clara para ter critérios de aceitação.

**Critério de saída:** Clarity Score ≥ 12/15 em todos os 5 elementos (problema, usuários, objetivos, sucesso, escopo).

**Artefatos:** `.claude/sdd/features/DEFINE_{FEATURE}.md` + `.claude/sdd/features/SEALED_{FEATURE}.md`

**Qualidade:** Use `/grill-me` (Claude adversarial) e `/judge` (GPT-4o externo) antes de avançar.

---

### Fase 2: Design (`/design`)

**O que produz:** Documento técnico com fluxo de arquitetura, responsabilidades por componente, file manifest com waves, estratégia de erro, e plano de rollback.

**Critério de entrada:** DEFINE com status "Approved". DESIGN é obrigatório se qualquer condição da seção abaixo for verdadeira.

**Critério de saída:** File manifest completo com Wave assignments. Arquitetura revisada por design-agent.

**Artefato:** `.claude/sdd/features/DESIGN_{FEATURE}.md`

---

### Fase 3: Build (`/build`)

**O que produz:** Código conforme o file manifest do DESIGN, executado por waves, com verificação após cada arquivo. Ao final, BUILD_REPORT com score de qualidade.

**Critério de entrada:** DESIGN aprovado. build-agent nunca lê arquivos SEALED.

**Critério de saída:** Quality Score ≥ 90%. Todos os arquivos do manifest criados e verificados.

**Artefatos:** Código + `.claude/sdd/reports/BUILD_REPORT_{FEATURE}.md`

---

### Fase 4: Ship (`/ship`)

**O que produz:** Arquivo SHIPPED com lições aprendidas, arquivamento de todos os docs, shadow score dos cenários SEALED, resultado do security gate.

**Critério de entrada:** Build completo com quality score ≥ 90%.

**Critério de saída:** Shadow score ≥ 90%. Security gate PASSED. Todos os artefatos no archive.

**Artefato:** `.claude/sdd/archive/{FEATURE}/SHIPPED_{DATE}.md`

---

## Quando DESIGN é Obrigatório

DESIGN pode ser pulado em features simples (DEFINE aprovado → BUILD direto). Mas é obrigatório se **qualquer** condição abaixo for verdadeira:

- [ ] Feature toca **mais de um serviço** (ex: Cloud Run + Firestore + BigQuery)
- [ ] Há **nova dependência externa** (nova API, novo SDK, nova conta de serviço)
- [ ] Envolve **mudança de schema** em dados persistentes (BigQuery, Firestore, Supabase)
- [ ] Introduz **padrão arquitetural novo** no projeto (ex: primeiro agente LLM, primeiro stream)
- [ ] Tem **implicação de segurança** (nova SA, novo secret, mudança de IAM, NEXT_PUBLIC_ secret)
- [ ] Estimativa de implementação **> 1 dia** de trabalho

Se nenhuma condição for verdadeira: documente no BUILD_REPORT `"DESIGN skipped: [razão]"`.

---

## SEALED Envelope Testing

O problema clássico de testes é que o desenvolvedor — consciente ou não — ajusta a implementação para passar nos testes visíveis. O resultado é um sistema que passa nos testes mas falha nos cenários reais.

**Como funciona:**

1. **define-agent** escreve `SEALED_{FEATURE}.md` como último passo do `/define`. O arquivo contém cenários Given/When/Then detalhados para cada critério de aceitação.
2. **build-agent** executa a build **sem ler** o arquivo SEALED. Regra: se vir um arquivo SEALED, pular.
3. **ship-agent** avalia os cenários SEALED após a build completa e calcula o shadow score.

**Formula do shadow score:**

```
shadow_score = (HIGH_passed × 0.60) + (MEDIUM_passed × 0.30) + (LOW_passed × 0.10)
```

**Mínimo para ship: 90%**

Se shadow_score < 90%: ship bloqueado → abrir `/iterate` para endereçar os gaps.

---

## Quality Gate

Antes de sugerir `/ship`, o build-agent calcula o quality score:

```
quality_score = (tests_pass_rate × 0.40) + (acceptance_pass_rate × 0.40) + (no_blockers × 0.20)
```

| Componente | Peso | Como medir |
|-----------|------|-----------|
| Unit/integration tests pass rate | 40% | Output do pytest / jest |
| Acceptance tests pass rate | 40% | AT Verification table no BUILD_REPORT |
| Zero blocking issues | 20% | Issues Encountered section |

**Se quality_score < 90%:** build-agent NÃO marca o build como completo nem sugere `/ship`. O status do BUILD_REPORT fica "⚠️ Blocked — quality gate not met".

---

## Security Gate

Integrado ao `/ship` como **Step 2b** — executado antes da avaliação do shadow score.

**Fluxo:**

```
ship-agent → security-orchestrator
                ├── secrets-scanner    (API keys, JWT, passwords, GCP SA JSON)
                ├── owasp-checker      (A01/A02/A03/A05/A09 por stack)
                └── infra-auditor      (IAM, RLS, Secret Manager, Terraform)
             ↓
          BLOCKED ou PASSED
```

**Decisão:**
- `total_critical ≥ 1` → **BLOCKED** — ship para imediatamente
- `total_high ≥ 3` → **BLOCKED**
- Demais → **PASSED** (com findings HIGH/LOW registrados no SECURITY_REPORT)

**Override:** Possível com justificativa não-vazia. Registrado no SECURITY_REPORT como "Accepted Risk".

**SECURITY_REPORT:** `.claude/sdd/reports/SECURITY_REPORT_{FEATURE}.md` — copiado para o archive no ship.

---

## `/judge` — Cross-Model Review

`/judge` envia o documento DEFINE, DESIGN ou BUILD_REPORT para **GPT-4o via OpenRouter** e recebe um veredito JSON estruturado. É uma revisão cega por um modelo diferente — sem acesso ao histórico do Claude.

**Diferença de `/grill-me`:**

| Comando | Modelo | Modo | O que avalia |
|---------|--------|------|-------------|
| `/grill-me` | Claude (Opus) | Diálogo adversarial | Qualidade — aprofunda o spec |
| `/judge` | GPT-4o (OpenRouter) | Revisão estruturada | Completude — todos os campos presentes |

**Quando usar:** `/grill-me` antes de avançar de fase (melhora qualidade). `/judge` para confirmar que não faltou nada (checa completude). Usados em sequência são blind peer review real.

**Requer:** `OPENROUTER_API_KEY` no environment.

```bash
python scripts/judge.py .claude/sdd/features/DEFINE_FEATURE.md
```

---

## Wave Scheduling no Build

O file manifest do DESIGN agrupa arquivos em waves com base em dependências. Arquivos dentro do mesmo wave não se dependem e podem ser criados em paralelo pelo build-agent.

| Wave | Conteúdo | Regra |
|------|---------|-------|
| 1 | Configs, schemas, constantes | Zero dependências |
| 2 | Lógica core, handlers | Depende de Wave 1 |
| 3 | Integrações, composição | Depende de Wave 2 |
| 4 | Testes | Depende de todas as waves de implementação |

**Regra de falha:** Se um arquivo falha na verificação após 3 tentativas, a wave para. O build-agent documenta o bloqueio no BUILD_REPORT e surfaceia ao usuário — não avança para a próxima wave.

---

## Stop Conditions & Escalation Rules

Cada workflow agent tem `stop_conditions` e `escalation_rules` no frontmatter YAML. São regras de governança que definem quando o agente deve parar de executar ou escalonar para o usuário.

**Por que existem:** Sem elas, o agente tende a continuar avançando mesmo quando encontra problemas — escrevendo docs incompletos, avançando de fase sem aprovação, ou executando waves com bloqueadores não resolvidos.

**Exemplo (build-agent):**

```yaml
stop_conditions:
  - "No DESIGN doc found: Stop immediately, redirect to design-agent"
  - "Quality score < 90% at BUILD_REPORT time: Do not mark complete or suggest /ship"

escalation_rules:
  - trigger: "Missing requirement discovered that blocks a wave"
    action: "Pause build — use /iterate to update DESIGN, then resume"
```

---

## Artefatos por Fase

| Artefato | Fase | Path | Quem escreve | Quem lê |
|----------|------|------|-------------|---------|
| `BRAINSTORM_{F}.md` | 0 | `sdd/features/` | brainstorm-agent | define-agent |
| `DEFINE_{F}.md` | 1 | `sdd/features/` | define-agent | design-agent, build-agent |
| `SEALED_{F}.md` | 1 | `sdd/features/` | define-agent | ship-agent **only** |
| `DESIGN_{F}.md` | 2 | `sdd/features/` | design-agent | build-agent |
| `BUILD_REPORT_{F}.md` | 3 | `sdd/reports/` | build-agent | ship-agent |
| `SECURITY_REPORT_{F}.md` | 4 | `sdd/reports/` | security-orchestrator | ship-agent |
| `SHIPPED_{DATE}.md` | 4 | `sdd/archive/{F}/` | ship-agent | future reference |

---

## Comandos de Suporte

| Comando | O que faz | Quando usar |
|---------|-----------|-------------|
| `/grill-me` | Adversarial Q&A no DEFINE — aprofunda qualidade | Antes de avançar para DESIGN |
| `/judge` | GPT-4o revisa DEFINE/DESIGN/BUILD — checa completude | Após /grill-me, para confirmar nada faltou |
| `/iterate` | Atualiza docs de qualquer fase com cascade awareness | Quando requisito muda mid-build |
| `/status` | Health dashboard — features ativas, git state, recomendações | Início de cada sessão |
| `/dev` | Dev Loop Level 2 — executa PROMPT.md com verificação | Features menores sem DESIGN formal |
| `/create-pr` | Cria PR com dual-review (CodeRabbit + Claude) | Antes de merge |
| `/sync-context` | Atualiza CLAUDE.md com estado atual do codebase | Após mudanças estruturais |
| `/meeting` | Extrai decisões/requisitos de notas de reunião | Após reunião de requisitos |

---

## Glossário

| Termo | Definição |
|-------|-----------|
| **Clarity Score** | Score 0-15 do DEFINE (3 pts × 5 elementos). Mínimo para avançar: 12/15. |
| **Shadow Score** | Score do SEALED test: `(HIGH×0.60) + (MED×0.30) + (LOW×0.10)`. Mínimo: 90%. |
| **Quality Score** | Score do BUILD: `(tests×0.40) + (ATs×0.40) + (no_blockers×0.20)`. Mínimo: 90%. |
| **Wave** | Grupo de arquivos sem dependências entre si — executados em paralelo no build. |
| **SEALED** | Arquivo de cenários holdout escrito pelo define-agent, avaliado só pelo ship-agent. |
| **Security Gate** | Step 2b do /ship — 4 agentes escaneiam o codebase. CRITICAL = BLOCKED. |
| **stop_conditions** | Regras no frontmatter do agente que definem quando ele deve parar de executar. |
| **escalation_rules** | Regras no frontmatter que definem quando e como escalar ao usuário. |
| **BUILD_REPORT** | Documento que registra o que foi implementado, desvios do DESIGN e quality score. |
| **SHIPPED doc** | Documento final de archive com lições aprendidas, shadow score e security result. |
| **Archive** | `.claude/sdd/archive/{FEATURE}/` — destino final de todos os artefatos shippados. |
| **Override** | Flag para bypassar um gate (security ou quality) com justificativa obrigatória. |

---

## Começando um Projeto Novo

1. **Clone o template** para o diretório do projeto
2. **Inicialize o workspace:**
   ```bash
   bash scripts/init-workspace.sh meu-projeto --stack fullstack
   ```
3. **Preencha `CLAUDE.md`** — descreva o problema de negócio e a stack real
4. **Execute `/status`** para verificar que o ambiente está correto
5. **Explore a primeira feature:**
   ```
   /brainstorm "quero construir X para resolver Y"
   ```
6. **Capture os requisitos:**
   ```
   /define .claude/sdd/features/BRAINSTORM_FEATURE.md
   ```
7. **Valide antes de avançar:**
   ```
   /grill-me .claude/sdd/features/DEFINE_FEATURE.md
   /judge .claude/sdd/features/DEFINE_FEATURE.md
   ```
8. **Design (se obrigatório):**
   ```
   /design .claude/sdd/features/DEFINE_FEATURE.md
   ```
9. **Build:**
   ```
   /build .claude/sdd/features/DESIGN_FEATURE.md
   ```
10. **Ship:**
    ```
    /ship .claude/sdd/features/DEFINE_FEATURE.md
    ```

---

## Referências

| Arquivo | Conteúdo |
|---------|----------|
| `sdd/architecture/ARCHITECTURE.md` | Visão arquitetural completa do SDD (42KB) |
| `sdd/architecture/TEMPLATE_ARCHITECTURE.md` | Extensões do template: SEALED, quality gate, security gate |
| `sdd/architecture/WORKFLOW_CONTRACTS.yaml` | Contratos máquina-legíveis entre fases e agentes |
| `rules/sdd-workflow.md` | Regras formais — critérios obrigatórios por fase |
| `rules/gcp-safety.md` | Comandos bloqueados no GCP |
| `rules/python-style.md` | Padrões de código Python |
| `scripts/judge.py` | CLI do /judge — validação offline via OpenRouter |
| `scripts/generate-agent-router.py` | Gera routing.json a partir dos agents |
| `scripts/init-workspace.sh` | Inicializa novo projeto a partir do template |
| `kb/shared/anti-patterns.md` | Anti-patterns cross-domain para todos os agentes |
