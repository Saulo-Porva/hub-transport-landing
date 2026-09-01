# Security KB — Index

> Entry point for all security scanning knowledge. Used by `security-orchestrator`, `secrets-scanner`, `owasp-checker`, and `infra-auditor`.

---

## Domain Overview

The Security KB supports the 4-agent security gate integrated into `/ship`. It covers:

| Domain | Scope | KB File |
|--------|-------|---------|
| Secrets detection | API keys, JWT tokens, passwords, GCP SA JSON, private keys | `concepts/secrets-patterns.md` |
| OWASP Top 10 | A01/A02/A03/A05/A09 checks per stack | `patterns/owasp-checks.md` |
| Fix suggestions | `current_code → fix_code` format, reason triples | `patterns/fix-suggestion-format.md` |
| Fast lookup | Regex patterns, severity codes, ID prefixes | `quick-reference.md` |

---

## KB Navigation

```text
.claude/kb/security/
├── index.md                      ← YOU ARE HERE
├── quick-reference.md            ← Fast lookup: regex, ID prefixes, thresholds
├── concepts/
│   └── secrets-patterns.md       ← Canonical secret patterns + false-positive exclusions
└── patterns/
    ├── owasp-checks.md           ← OWASP A01/A02/A03/A05/A09 per stack
    └── fix-suggestion-format.md  ← Standard fix output format
```

---

## Which File to Load by Agent

| Agent | Load These KB Files |
|-------|---------------------|
| `secrets-scanner` | `quick-reference.md` + `concepts/secrets-patterns.md` + `patterns/fix-suggestion-format.md` |
| `owasp-checker` | `quick-reference.md` + `patterns/owasp-checks.md` + `patterns/fix-suggestion-format.md` |
| `infra-auditor` | `quick-reference.md` + `patterns/owasp-checks.md` (A05 section) + `patterns/fix-suggestion-format.md` |
| `security-orchestrator` | `quick-reference.md` only (delegates checks to scanners) |

---

## Scanner Output Contract

All three scanners emit **structured YAML** that the orchestrator aggregates:

```yaml
scanner_name: string      # "secrets-scanner" | "owasp-checker" | "infra-auditor"
status: string            # "passed" | "findings_found" | "no_source_files" | "error"
findings:
  - id: string            # SS-NNN | OW-NNN | IA-NNN
    severity: string      # CRITICAL | HIGH | MEDIUM | LOW
    title: string
    file: string          # relative path from repo root
    line: integer
    current_code: string
    fix_code: string
    reason: string
    owasp_ref: string | null
```

---

## Stacks Covered

| Stack | Where Encountered |
|-------|-------------------|
| Next.js 15 (TypeScript) | `src/**/*.ts`, `src/**/*.tsx`, `*.js`, `next.config.*` |
| Python / Cloud Run | `**/*.py`, `Dockerfile`, `requirements.txt` |
| Supabase | SQL migrations in `supabase/migrations/**/*.sql` |
| GCP / Terraform | `infra/**/*.tf`, `**/*.yaml` (Cloud Build, IAM) |

---

## Severity Reference

| Severity | Meaning | Ship Impact |
|----------|---------|-------------|
| CRITICAL | Immediate exploitable risk | BLOCKED — must fix before ship |
| HIGH | Significant risk, likely exploitable | Warning — review before next release |
| MEDIUM | Moderate risk, limited scope | Warning — track and fix |
| LOW | Minor issue, best practice | Informational |
