# Hub_Transport_Landing (S.H.S)

> Founder-led software studio landing page. Problem-first positioning: "Show us the bottleneck. We'll build the system." Ships TruckPilot (WhatsApp fleet-compliance co-pilot) among 7 shipped systems.

---

## Project Context

**Business Problem:** The site used to sell TruckPilot as a single product. It now sells one competency — turning operational bottlenecks (manual work, spreadsheets, no visibility, WhatsApp overload) into software — with TruckPilot as one of 7 proof points, 3 of them featured.

**Solution:** A problem-first home page (hero → problem selector → results band → 3 featured + 4 "more systems" case tier → open funnel → demo form) links out to 7 individual case-study pages, each following Problem → Before → System → How It Works → Result → Technology → CTA. This page's job is to convert visitors into inquiries, not just demo requests.

**Stack:** Static multi-page HTML + Tailwind CDN + vanilla JS (i18n, form handling), no build step, no backend. Deployed on Netlify. Form submissions via FormSubmit.co. Not a Python/cloud project — most of the KB/agents below (GCP, Terraform, Spark, BigQuery...) target a *product's* backend stack, not this repo, and won't be exercised here until/unless backend code lands in this project.

---

## Architecture Overview

```text
Visitor → index.html (light/editorial theme, EN/IT/DE i18n)
        → hero ("Show us the bottleneck. We'll build the system.") → problem selector → results band
        → about (founder-led studio) → how-it-works track
        → solutions: 3 featured (Fleet Control, Workforce, AI Real Estate) + 4 "more systems"
        → open funnel ("Don't see your exact problem here?")
        → "See the case study →" per card → case-{name}.html (Problem → Before → System → How It Works → Result → Technology → CTA)
        → #demo form (on index.html, generic problem-based fields) → FormSubmit.co → saulohs@icloud.com
        → success.html (fallback confirmation)

Shared: styles.css (used by all 8 pages)

Hosting: Netlify (netlify.toml — publish "." + security headers)
```

| Layer | Technology | Purpose |
|-------|------------|---------|
| Hosting | Netlify | Static hosting, auto-deploy on push |
| Frontend | HTML + Tailwind CDN + vanilla JS | Multi-page marketing site, client-side i18n on index.html + case-truckpilot.html |
| Lead capture | FormSubmit.co | Demo request form → email, no backend needed |
| Assets | `shs-logo.png` | Footer brand identity (S.H.S) |

---

## Project Structure

```text
Hub_Transport_Landing/
├── index.html                  # Hero, problem selector, results band, About, How-it-works, Solutions (3 featured + 4 more), funnel, Contact — EN/IT/DE, light theme
├── case-truckpilot.html        # More systems — full detail (7 WhatsApp workflows, regulatory, FAQ) — EN/IT/DE, light theme
├── case-fleet-control.html     # Featured case — EN only (renamed from Rapportini)
├── case-shift-scheduling.html  # Featured case, displayed as "Workforce" — EN only
├── case-real-estate.html       # Featured case, displayed as "AI Real Estate" — EN only
├── case-health-anywhere.html   # More systems — EN only
├── case-contabile.html         # More systems — EN only
├── case-workout.html           # More systems — EN only
├── styles.css                  # Shared CSS, used by all 8 pages above
├── success.html                # Fallback success page for form submission
├── shs-logo.png                # Footer logo asset
├── netlify.toml                # Netlify build/publish + security headers
├── README.md                   # Deploy instructions + content editing guide
├── .claude/                    # Claude Code ecosystem (this directory)
│   ├── agents/                 # 40+ specialized agents (mostly cloud/data — see note above)
│   ├── commands/                # Slash commands
│   ├── hooks/                    # Security + automation hooks
│   ├── kb/                        # Knowledge Base (24 domains)
│   ├── rules/                      # Coding + workflow rules
│   └── sdd/                    # Spec-Driven Development artifacts
└── (no pyproject.toml — no Python code in this repo yet)
```

---

## Development Workflows

### AgentSpec 4.1 — Spec-Driven Development (SDD)

```text
/brainstorm → /define → [/grill-me] → /design → [/grill-me] → /build → /ship
```

| Command | Phase | Purpose |
|---------|-------|---------|
| `/brainstorm` | 0 | Explore ideas through dialogue (optional) |
| `/define` | 1 | Capture and validate requirements |
| `/grill-me` | 1.5 / 2.5 | Stress-test DEFINE or DESIGN before advancing (optional) |
| `/design` | 2 | Create architecture and specification |
| `/build` | 3 | Execute implementation with verification |
| `/ship` | 4 | Archive with lessons learned |
| `/iterate` | Any | Update documents when requirements change |

**Artifacts:** `.claude/sdd/features/` → `.claude/sdd/archive/`

### Dev Loop — Level 2 Agentic Development

```bash
/dev "I want to build a date parser utility"
/dev tasks/PROMPT_DATE_PARSER.md
/dev tasks/PROMPT_DATE_PARSER.md --resume
```

Use for: KB building, prototypes, utilities, single-file features.

---

## Available Agents

| Category | Agents | Use When |
|----------|--------|----------|
| **Workflow** | brainstorm, define, design, build, ship, iterate | SDD feature development |
| **Code Quality** | code-reviewer, code-cleaner, python-developer, test-generator, dual-reviewer | Code review and improvement |
| **AI/ML** | llm-specialist, genai-architect, ai-prompt-specialist, ai-data-engineer | LLM prompts, AI systems |
| **Data Engineering** | spark-*, lakeflow-*, medallion-architect | Spark, Databricks, Medallion |
| **AWS** | aws-deployer, lambda-builder, aws-lambda-architect, ci-cd-specialist | AWS deployments |
| **Frontend** | nextjs-specialist, ui-designer, ux-specialist, marketing-specialist, product-quality-auditor, landing-parity-auditor | Next.js, UI/UX design, marketing copy, landing/case-study review |
| **Infra/Deploy** | supabase-specialist, vercel-specialist, infra-deployer | Supabase, Vercel, GCP deploy |
| **Communication** | adaptive-explainer, meeting-analyst, the-planner | Documentation, planning |
| **Exploration** | codebase-explorer, kb-architect | Codebase exploration, KB creation |
| **Dev** | prompt-crafter, dev-loop-executor | Dev Loop workflow |

---

## Commands

| Command | Purpose |
|---------|---------|
| `/brainstorm` | Explore ideas through collaborative dialogue |
| `/define` | Capture and validate requirements |
| `/grill-me` | Stress-test DEFINE or DESIGN before advancing |
| `/design` | Create technical architecture |
| `/build` | Execute implementation |
| `/ship` | Archive completed features |
| `/iterate` | Update documents mid-stream |
| `/dev` | Dev Loop for structured iteration |
| `/create-kb` | Create knowledge base domains |
| `/review` | Code review workflow |
| `/create-pr` | Create pull requests |
| `/memory` | Save session insights |
| `/sync-context` | Update CLAUDE.md with project context |
| `/readme-maker` | Generate comprehensive README |

---

## Knowledge Base (24 domains)

| Domain | Purpose |
|--------|---------|
| **pydantic** | Data validation, Pydantic v2 patterns |
| **gcp** | GCP serverless — Cloud Run, Pub/Sub, GCS, BigQuery |
| **gemini** | Gemini multimodal LLM, Vertex AI |
| **langfuse** | LLMOps observability |
| **terraform** | Infrastructure as Code |
| **terragrunt** | Multi-environment IaC orchestration |
| **crewai** | Multi-agent AI orchestration |
| **openrouter** | Unified LLM API gateway / fallback |
| **agents** | LLM agent design patterns (ReAct, hooks, eval) |
| **bigquery** | Google BigQuery — schema, queries, streaming |
| **lakeflow** | Databricks Lakeflow / DLT pipelines |
| **medallion** | Medallion architecture Bronze/Silver/Gold |
| **spark** | Apache Spark / PySpark |
| **whatsapp** | WhatsApp Business API (Meta) |
| **gcp-safety** | GCP destructive ops — what to block |
| **aws-safety** | AWS destructive ops — what to block |
| **azure-safety** | Azure destructive ops — what to block |
| **ui-design** | Visual design — color, typography, layout, Tailwind |
| **ux** | UX — accessibility, user flows, field-worker/mobile UX |
| **nextjs** | Next.js 15 App Router — RSC, Server Actions, caching, middleware |
| **supabase** | Supabase with Next.js — RLS, Auth SSR, client types, debugging |
| **vercel** | Vercel deploys — environments, env vars, function logs, build config |
| **prompt-engineering** | Chain-of-thought, few-shot, structured extraction, system prompts |
| **security** | OWASP checks, secrets detection, fix-suggestion format |
| **marketing-copy** | Commercial copywriting — headlines, positioning, social proof, case-study/CTA structure |

---

## Hooks (Active)

| Hook | Trigger | Purpose |
|------|---------|---------|
| `pre-bash-cloud-safety.js` | Bash | Blocks destructive GCP + AWS + Azure commands |
| `pre-bash-secrets.js` | Bash | Detects hardcoded secrets in commands |
| `pre-bash-sqz.js` | Bash | Compresses Bash output for Claude context |
| `post-write-ruff.js` | Write/Edit | Auto-runs ruff lint+format on Python files |
| `stop-loop.js` | Stop | Blocks stop if tests fail or TODOs remain |
| `post-stop-notify.js` | Stop | Desktop notification when Claude finishes |

---

## Coding Standards

### Python 3.11+

- **Linter/Formatter:** Ruff (line-length 100, select E/F/I/UP/B/SIM)
- **Validation:** Pydantic v2 for all data models
- **Testing:** pytest with `-v --tb=short`
- **Type Hints:** Required on all function signatures
- **Package Manager:** pyproject.toml with hatchling

### Rules

- `rules/python-style.md` — Python coding standards
- `rules/sdd-workflow.md` — SDD phase rules and artifact requirements
- `rules/cloud-safety.md` — Cloud safety and deployment patterns

---

## Environment Variables

None. This is a fully static site — the demo form posts directly to FormSubmit.co from the browser, no server-side secrets involved.

---

## Getting Help

- **SDD Workflow:** `.claude/sdd/_index.md`
- **SDD Templates:** `.claude/sdd/templates/`
- **SDD Examples:** `.claude/sdd/examples/`
- **Dev Loop:** `.claude/dev/_index.md`
- **Agents:** `.claude/agents/`
- **KB Index:** `.claude/kb/_index.yaml`

---

## Version History

| Date | Changes |
|------|---------|
| 2026-09-01 | Initial setup from claude-project-template (full .claude/ scaffold — agents/KB target the future backend stack, not this static landing repo) |
| 2026-09-02 | Problem-first repositioning: light/editorial theme across all 8 pages, new hero + problem selector + results band, 3-featured/4-more-systems case tier, Rapportini renamed to Fleet Control, Before/Technology sub-steps added to every case page, demo form genericized, open funnel section added |
