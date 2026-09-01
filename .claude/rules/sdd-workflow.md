# SDD Workflow Rules — Spec-Driven Development

Rules for the 5-phase AgentSpec 4.1 workflow. These govern when to use each phase and what each artifact must contain.

---

## Phase Entry Criteria

| Phase | Command | Enter When |
|-------|---------|-----------|
| Brainstorm | `/brainstorm` | Idea is vague, tradeoffs unclear, stakeholder alignment needed |
| Define | `/define` | Requirements clear enough to specify acceptance criteria |
| Design | `/design` | DEFINE approved, architecture decisions needed |
| Build | `/build` | DESIGN approved, implementation ready to start |
| Ship | `/ship` | Build complete, tests pass, ready to archive |

**Rule:** Never skip to `/build` without a DESIGN doc. Implementation without spec = tech debt.

---

## DESIGN Required If (OpenSpec — Conditional Phase)

> A fase DESIGN é obrigatória se qualquer condição abaixo for verdadeira. Caso contrário, pode ir direto para BUILD com um DEFINE aprovado.

- [ ] A feature toca **mais de um serviço** (ex: Cloud Run + Firestore + BigQuery)
- [ ] Há **nova dependência externa** (nova API, novo SDK, nova conta de serviço)
- [ ] Envolve **mudança de schema** em dados persistentes (BigQuery, Firestore, Pub/Sub)
- [ ] Introduz **padrão arquitetural novo** no projeto (ex: primeiro agente LLM, primeiro stream)
- [ ] Tem **implicação de segurança** (nova SA, novo secret, mudança de IAM)
- [ ] Estimativa de implementação > **1 dia** de trabalho

**Se nenhuma condição acima for verdadeira:** DEFINE aprovado → BUILD direto, sem DESIGN formal.  
**Documentar a decisão:** Em features simples, adicionar ao BUILD_REPORT: `"DESIGN skipped: [razão]"`.

---

## Artifact Requirements

### DEFINE_*.md must contain
- Business problem and user story
- Acceptance criteria (measurable, testable)
- Out of scope (explicit)
- Data contracts (input/output schemas)
- Dependencies on other features

### DESIGN_*.md must contain
- Architecture diagram or flow
- Component responsibilities
- Data flow between components
- Error handling strategy
- BigQuery/Firestore schema changes (if any)
- Rollback plan

### BUILD_REPORT_*.md must contain
- What was implemented vs the spec
- Deviations from DESIGN and why
- Test coverage
- Known limitations
- Follow-up tasks

---

## Active Features Policy

- `.claude/sdd/features/` = in-progress only
- Never accumulate more than 3 active features simultaneously
- Ship or archive before starting new features
- Features blocked >2 weeks must be explicitly re-triaged

---

## LLM-Heavy Features (extra rules)

When the feature involves LLM extraction, classification, or generation:

1. **DESIGN must include prompt draft** — not "TBD"
2. **DESIGN must include Agent Communication Contract** — tabela sender/receiver/content/metadata para cada step
3. **BUILD must include LangFuse trace IDs** for validation runs
4. **BUILD must include EvalTask results** — `EvalResult.summary()` no BUILD_REPORT
5. **SHIP must include accuracy metric** (≥90% threshold for extraction)
6. **Never ship** an LLM feature without at least 10 real-world test cases

### Evaluation Framework (BUILD obrigatório)

Todo BUILD de feature LLM deve incluir um `EvalTask` com:

```python
# Estrutura mínima — ver kb/agents/patterns/agent-evaluation.md
EvalTask(
    task_name="{feature}_eval_v1",
    agent_name="{agent}",
    threshold=0.90,           # 0.85 para classificação
    cases=[...],              # mínimo 10 casos reais
)
```

**Thresholds por tipo:**

| Feature | Threshold | Bloqueante? |
|---------|-----------|-------------|
| Extração de campos de documento | ≥ 90% field_accuracy | Sim |
| Classificação de documento | ≥ 85% precision | Sim |
| NLP / extração de intenção | ≥ 80% recall | Recomendado |

**Referência:** `.claude/kb/agents/patterns/agent-evaluation.md`

---

## Data Pipeline Features (extra rules)

When the feature modifies BigQuery schema or Pub/Sub topics:

1. **DESIGN must include migration plan** — how existing data is handled
2. **BUILD must include schema diff** — before/after table definition
3. **Never change** a column type or remove a column without a DEFINE
4. **Test in dev** BigQuery before any schema change on prod

### Schema Drift Rule (Design as Code)

> Inspirado no princípio "Design as Code" do Pencil.dev: schemas Pydantic são os design tokens do pipeline. Quando eles derivam do contrato especificado, o débito técnico de spec se acumula.

**Regra:** Qualquer PR que modifica um arquivo em `shared/schemas/` para uma feature já shippada **deve** incluir um novo `DEFINE_SCHEMA_UPDATE_{FEATURE}.md` antes de merge.

**Checklist de schema drift:**

- [ ] O campo removido ou renomeado quebra consumidores downstream (BigQuery, Pub/Sub)?
- [ ] O DESIGN doc original reflete a mudança?
- [ ] O BUILD_REPORT da feature original registra a divergência?
- [ ] Os testes de integração cobrem o novo contrato?

**O que nunca fazer:**
- Alterar tipo de campo em schema existente sem DEFINE (silently breaking)
- Adicionar campo `required` sem migration plan para dados históricos
- Renomear campo sem alias de compatibilidade temporário

---

## Agent Usage in Build Phase

```markdown
# ✓ Correct agent references in PROMPT.md
- [ ] @python-developer: Implement WebhookHandler class
- [ ] @test-generator: Add unit tests for intent_classifier
- [ ] @code-reviewer: Review changes before ship

# ✗ Wrong — too vague
- [ ] @python-developer: Build the feature
```

---

## Archive Policy

Shipped features move to `.claude/sdd/archive/FEATURE_NAME/`:
- All phase docs (BRAINSTORM, DEFINE, DESIGN)
- SEALED_{FEATURE}.md
- BUILD_REPORT
- SHIPPED_DATE.md with lessons learned and patterns extracted
- Lessons learned are mandatory — minimum 3 bullet points

---

## Sealed Envelope Testing

Inspired by the "dark factory" pattern — acceptance scenarios the builder never sees, preventing implementation from being gamed toward visible tests.

### How it works

1. **define-agent** writes `SEALED_{FEATURE}.md` as the **last step** of `/define`
   - Contains verbose Given/When/Then scenarios for every AT
   - File is stored in `.claude/sdd/features/` alongside DEFINE
2. **build-agent** executes the build **without reading** `SEALED_*.md`
   - Rule: if you see a SEALED file, skip it
3. **ship-agent** evaluates the sealed scenarios after build, records shadow score
   - If shadow_score < 90%: block ship, open `/iterate` to address gaps

### Artifact

| File | When created | Who reads |
|------|-------------|-----------|
| `SEALED_{FEATURE}.md` | End of `/define` | ship-agent only |

### Shadow Score

```
shadow_score = (HIGH_passed * 0.60) + (MEDIUM_passed * 0.30) + (LOW_passed * 0.10)
```
Minimum to ship: **90%**

---

## Wave Scheduling (Build Phase)

Files in the DESIGN manifest are grouped into waves. Files within the same wave have no interdependencies and can be delegated in parallel by build-agent.

### Wave Assignment Rules

| Wave | Content | Rule |
|------|---------|------|
| 1 | Configs, schemas, constants | Zero dependencies |
| 2 | Core logic, handlers | Depends on Wave 1 |
| 3 | Integrations, composition | Depends on Wave 2 |
| 4 | Tests | Depends on all implementation waves |

**Rule:** Wave N only starts after **all files in Wave N-1 pass verification**.
If a file fails after 3 retries, the wave is blocked — stop, document, surface to user.

### In the DESIGN template
The File Manifest includes a `Wave` column. design-agent assigns waves based on dependency analysis.

---

## Quality Score Gate

Every BUILD must calculate and report a quality score before proceeding to `/ship`.

```
quality_score = (tests_pass_rate * 0.40) + (acceptance_pass_rate * 0.40) + (no_blockers * 0.20)
```

**Minimum to ship:** 90%

| Component | Weight | How to measure |
|-----------|--------|----------------|
| Unit/Integration tests pass rate | 40% | pytest output |
| Acceptance tests pass rate | 40% | AT Verification table in BUILD_REPORT |
| Zero blocking issues | 20% | Issues Encountered section |

**Rule:** build-agent must NOT update document statuses or present the `/ship` next step if quality_score < 90%.

---

## Phase Checkpoints (Git Tags)

At each phase completion, create a lightweight git tag so the phase can be restored if subsequent work goes wrong.

| Phase | Tag format | When |
|-------|-----------|------|
| After /define | `sdd/{FEATURE}/defined` | define-agent completes the DEFINE file |
| After /design | `sdd/{FEATURE}/designed` | design-agent completes the DESIGN file |
| After /build | `sdd/{FEATURE}/built` | build-agent completes BUILD_REPORT |
| After /ship | `sdd/{FEATURE}/shipped` | ship-agent archives everything |

```bash
# Example — run after each phase:
git tag sdd/USER_NOTIFICATIONS/defined
git tag sdd/USER_NOTIFICATIONS/designed
git tag sdd/USER_NOTIFICATIONS/built
git tag sdd/USER_NOTIFICATIONS/shipped
```

**Rule:** Tags are optional but strongly recommended. If a build goes wrong, `git checkout sdd/{FEATURE}/designed` restores the last stable spec state without losing prior work.
