# Security KB — Quick Reference

> Fast lookup card for scanners. Load this file first before any security scan.

---

## Finding ID Prefixes

| Scanner | Prefix | Example |
|---------|--------|---------|
| `secrets-scanner` | `SS-` | `SS-001`, `SS-002` |
| `owasp-checker` | `OW-` | `OW-001`, `OW-002` |
| `infra-auditor` | `IA-` | `IA-001`, `IA-002` |

Increment sequentially per scan run. Reset to 001 each invocation.

---

## Severity Decision Rules

| Condition | Severity |
|-----------|----------|
| Hardcoded secret (live/production key) | CRITICAL |
| Test fixture with real-looking secret pattern | HIGH |
| OWASP A01/A02/A03 code pattern | CRITICAL |
| OWASP A05/A09 pattern | HIGH |
| RLS disabled on table | CRITICAL |
| Secret in env var (not Secret Manager) | HIGH |
| IAM `roles/editor` or `roles/owner` on SA | CRITICAL |
| `force_destroy = true` on GCS/SQL | CRITICAL |
| `console.log` with PII fields | HIGH |
| Missing Content-Security-Policy header | MEDIUM |

---

## Secret Pattern Quick Regex

```text
Stripe live key:     sk_live_[a-zA-Z0-9]{20,}
Stripe test key:     sk_test_[a-zA-Z0-9]{20,}
Google API key:      AIza[0-9A-Za-z\-_]{35}
AWS Access Key:      AKIA[0-9A-Z]{16}
JWT token:           eyJ[A-Za-z0-9\-_]+\.eyJ[A-Za-z0-9\-_]+\.[A-Za-z0-9\-_]+
PostgreSQL DSN:      postgresql://[^:]+:[^@]+@
MongoDB URI:         mongodb://[^:]+:[^@]+@
GCP SA type field:   "type":\s*"service_account"
Private key PEM:     -----BEGIN (RSA |EC )?PRIVATE KEY-----
Hardcoded password:  (?i)password\s*[=:]\s*["'][^"']{8,}["']
```

---

## Source File Discovery (all scanners)

```text
INCLUDE:
  **/*.py
  **/*.ts  **/*.tsx
  **/*.js  **/*.jsx
  **/*.yaml  **/*.yml
  **/*.json  (excluding package-lock.json, yarn.lock)
  **/*.env*
  **/*.pem  **/*.key
  **/terraform/**/*.tf
  supabase/migrations/**/*.sql

EXCLUDE:
  node_modules/**
  .next/**
  __pycache__/**
  .git/**
  dist/**
  build/**
  .venv/**
```

---

## False-Positive Exclusions (secrets-scanner)

Never flag as CRITICAL — downgrade to HIGH or skip:

```text
File path patterns → downgrade to HIGH (not CRITICAL):
  test_*.py
  *.test.ts  *.test.js  *.spec.ts  *.spec.js
  **/__tests__/**
  **/*.test.*  **/*.spec.*

Value patterns → skip entirely (not a real secret):
  "YOUR_API_KEY"
  "xxx"  "XXXX"
  "placeholder"
  "<redacted>"  "<YOUR_KEY>"
  "example"  "EXAMPLE"
  "changeme"  "todo"
  value length < 8 characters

Comment patterns → skip:
  Line contains: # test, # fixture, # example, # placeholder, // test, // example
```

---

## OWASP Quick Codes

| Code | Name | Stack Focus |
|------|------|-------------|
| A01:2021 | Broken Access Control | Next.js Server Actions, Supabase RLS |
| A02:2021 | Cryptographic Failures | Password hashing, secrets in env |
| A03:2021 | Injection | SQL injection, shell injection, eval() |
| A05:2021 | Security Misconfiguration | IAM, CORS, CSP headers |
| A09:2021 | Security Logging Failures | console.log PII, missing audit trails |

---

## Status Values

```yaml
status: "passed"           # 0 findings
status: "findings_found"   # ≥1 finding (any severity)
status: "no_source_files"  # no scannable files in repo
status: "error"            # scanner encountered an exception
```

---

## Orchestrator Decision

```text
total_critical = SUM of CRITICAL findings across all 3 scanners

override present AND justification non-empty → PASSED (with override note)
total_critical >= 1                          → BLOCKED
total_critical == 0                          → PASSED
```
