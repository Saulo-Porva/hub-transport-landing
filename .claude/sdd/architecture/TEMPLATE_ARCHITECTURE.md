# claude-project-template Architecture

> Template-specific extensions to AgentSpec 4.2. Read this alongside ARCHITECTURE.md for the full picture.

---

## What This Template Adds Over Base AgentSpec

| Feature | Base AgentSpec | This Template |
|---------|---------------|---------------|
| SEALED holdout testing | ❌ | ✅ holdout + shadow score (HIGH×0.60, MED×0.30, LOW×0.10) |
| `/grill-me` adversarial review | ❌ | ✅ Before DESIGN |
| Security gate at /ship | ❌ | ✅ 4-agent: secrets-scanner + owasp-checker + infra-auditor + orchestrator |
| Quality score gate (90%) | ❌ | ✅ tests×0.40 + ATs×0.40 + no_blockers×0.20 |
| `stop_conditions` in agents | ✅ (origin) | ✅ adapted for our SDD rules |
| Full-stack agents (Next.js, Supabase, Vercel) | ❌ | ✅ |
| Dev Loop Level 2 (`/dev`) | ❌ | ✅ |
| Dual reviewer (CodeRabbit + Claude) | ❌ | ✅ |
| Auto-memory system (4 types) | ❌ | ✅ |
| `/judge` cross-model second opinion | ✅ (origin) | ✅ adapted with our thresholds |
| BI agent (Looker Studio) | ❌ | ✅ |
| DataOps / self-healing (CrewAI) | ❌ | ✅ |

---

## Phase Contracts (Template-Specific Thresholds)

### Phase 1 → Phase 2

```
Transition gate: Clarity score ≥ 12/15

Clarity scoring (define-agent):
  Problem statement:  0-3 pts
  Users / personas:   0-3 pts
  Goals (measurable): 0-3 pts
  Success criteria:   0-3 pts
  Scope boundaries:   0-3 pts
  Total:             15 pts
  Minimum:           12 pts (80%)

Quality tools available:
  /grill-me  → adversarial dialogue (Claude asks hardest questions)
  /judge     → cross-model review (GPT-4o reads spec, returns PASS/FAIL)
```

### Phase 3 → Phase 4

```
Transition gate: quality_score ≥ 90%

quality_score = (tests_pass_rate × 0.40)
              + (acceptance_pass_rate × 0.40)
              + (no_blockers × 0.20)
```

### Phase 4 Ship (Template additions)

```
Step 2b: Security gate (before shadow score)
  security-orchestrator → [secrets-scanner, owasp-checker, infra-auditor]
  Result: BLOCKED (critical_count ≥ 1) or PASSED

SEALED evaluation:
  shadow_score = (HIGH_passed × 0.60)
               + (MEDIUM_passed × 0.30)
               + (LOW_passed × 0.10)
  Minimum to ship: 90%
```

---

## Security Gate Detail

```
/ship
  └─ Step 2b: security-orchestrator
       ├─ secrets-scanner
       │    Patterns: API keys (Stripe/Google/AWS/GitHub/Slack)
       │              JWT tokens
       │              Connection strings (PostgreSQL/MySQL/MongoDB/Redis)
       │              GCP Service Account JSON
       │              Hardcoded passwords
       │              Generic base64 secrets
       │    Output: findings[] with YAML schema (id, severity, file, line, current_code, fix_code)
       │
       ├─ owasp-checker
       │    A01: Broken Access Control (Server Actions without auth, RLS disabled)
       │    A02: Crypto Failures (MD5/SHA1 passwords, NEXT_PUBLIC_ secrets)
       │    A03: Injection (f-string SQL, shell=True, eval(), dangerouslySetInnerHTML)
       │    A05: Misconfig (missing CSP, --allow-unauthenticated, roles/editor)
       │    A09: Logging (console.log PII, print() in Cloud Run)
       │    Output: findings[] per stack (Next.js .ts/.tsx, Python .py, SQL, Terraform .tf)
       │
       └─ infra-auditor
            IAM: roles/editor or roles/owner on service accounts
            Supabase RLS: DISABLE, missing ENABLE, no policies
            Secret Manager: os.environ with sensitive key names
            Terraform: force_destroy, deletion_protection=false, prevent_destroy missing
            Output: findings[] with severity CRITICAL/HIGH/MEDIUM/LOW

  Decision logic:
    total_critical = sum of CRITICAL findings across all scanners
    if total_critical >= 1: BLOCKED → stop ship, surface SECURITY_REPORT
    else: PASSED → continue to SEALED shadow score evaluation

  Override: user provides written justification → risk accepted, documented in report
```

---

## Agent Model Allocation

| Phase | Agent | Model | Reason |
|-------|-------|-------|--------|
| 0 | brainstorm-agent | opus | Nuanced dialogue, creative exploration |
| 1 | define-agent | opus | Requirements understanding, clarity scoring |
| 2 | design-agent | opus | Architectural decisions, trade-off analysis |
| 3 | build-agent | sonnet | Fast, accurate code generation |
| 3 | iterate-agent | sonnet | Balanced speed for cross-phase updates |
| 4 | ship-agent | haiku | Fast archival, low complexity |
| 4 | security-orchestrator | sonnet | Precise pattern matching |

---

## /judge vs /grill-me

These are complementary, not competing:

```
/grill-me → Claude asks the hardest questions about the DEFINE
             "Is this scope too wide?" "What breaks this assumption?"
             Purpose: improve quality of spec before design starts
             When: after /define, before /design
             Mechanism: Claude-to-Claude adversarial dialogue

/judge    → GPT-4o (via OpenRouter) reads the document and judges it
             "Is the spec complete? Any gaps? Any logical contradictions?"
             Purpose: independent second opinion from a different model
             When: any phase, especially before advancing
             Mechanism: cross-model API call (requires OPENROUTER_API_KEY)

Use both for critical features. Use only /grill-me for routine features.
```

---

## SEALED Testing Flow

```
define-agent (Phase 1)
  └─ writes SEALED_{FEATURE}.md to .claude/sdd/features/
       Contains: verbose Given/When/Then for every AT
       NOT read by build-agent

build-agent (Phase 3)
  └─ sees SEALED file in directory
  └─ MUST SKIP IT — stop_condition enforces this

ship-agent (Phase 4)
  └─ reads SEALED_{FEATURE}.md
  └─ evaluates each scenario against what was built
  └─ computes shadow_score
  └─ if shadow_score < 90%: block ship, open /iterate to address gaps
  └─ copies SEALED to archive alongside other phase docs
```

---

## Wave Scheduling

```
DESIGN file manifest example:

| File | Action | Wave | Dependencies |
|------|--------|------|-------------|
| lib/config.ts | CREATE | 1 | None |
| lib/types.ts | CREATE | 1 | None |
| lib/db.ts | CREATE | 2 | lib/types.ts |
| api/handler.ts | CREATE | 2 | lib/config.ts, lib/types.ts |
| components/Form.tsx | CREATE | 3 | api/handler.ts |
| tests/handler.test.ts | CREATE | 4 | api/handler.ts |

Build execution:
  Wave 1: [lib/config.ts, lib/types.ts] → parallel ✓
  Wave 2: [lib/db.ts, api/handler.ts]   → parallel ✓ (after Wave 1 complete)
  Wave 3: [components/Form.tsx]         → sequential (after Wave 2 complete)
  Wave 4: [tests/]                      → after all implementation waves

Circuit breaker:
  File fails verification → retry up to 3 times
  After 3 failures → wave BLOCKED → surface to user → stop build
```

---

## Glossary

| Term | Definition |
|------|-----------|
| **Clarity score** | 0-15 score on a DEFINE doc. ≥12 required to advance to DESIGN. |
| **Shadow score** | Post-build evaluation: HIGH×0.60 + MED×0.30 + LOW×0.10. ≥90% required to ship. |
| **Quality score** | Pre-ship gate: tests×0.40 + ATs×0.40 + no_blockers×0.20. ≥90% required. |
| **Wave** | Group of files in DESIGN manifest with no inter-dependencies — can be built in parallel. |
| **SEALED file** | Holdout scenarios written by define-agent, read only by ship-agent. |
| **Security gate** | Step 2b in /ship — 3 scanners → orchestrator → BLOCKED or PASSED. |
| **stop_condition** | Agent frontmatter rule — when agent must halt and surface to user. |
| **escalation_rule** | Agent frontmatter rule — when agent must pause and ask for direction. |
| **AT** | Acceptance Test — testable Given/When/Then in DEFINE, evaluated via SEALED. |
| **/judge** | Cross-model second opinion: sends doc to GPT-4o via OpenRouter, returns PASS/FAIL JSON. |
| **/grill-me** | Adversarial Claude-to-Claude dialogue to stress-test a DEFINE doc. |
| **Dev Loop Level 2** | `/dev` command — executes PROMPT_*.md files with circuit breakers and session recovery. |
