# Shared Anti-Patterns

> Cross-domain anti-patterns applicable to all agents and all stacks.
> Load this file when reviewing code, designing systems, or validating specs.

---

## SDD Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Skipping DESIGN for multi-service features | No architecture = uncoordinated implementation, breaking changes discovered late | Follow DESIGN mandatory criteria checklist |
| Writing DEFINE vagueness past Clarity Score 12/15 | Vague specs → vague builds → rework | Run `/grill-me` before advancing to DESIGN |
| Reading SEALED_*.md as build-agent | Defeats the holdout test — implementation tuned to visible tests | Skip SEALED files; ship-agent evaluates them post-build |
| Marking tasks complete before verification | False progress, cascading failures in wave scheduling | Verify each file compiles/passes lint before marking done |
| Opening >3 active features simultaneously | Context overload, none ships cleanly | Ship or archive before starting new feature |
| Amending published commits | Destroys git history for collaborators | Create new commits; --amend only for unpushed work |
| Bypassing quality score gate (<90%) | Ships incomplete work disguised as done | Fix the failing ATs; do not inflate scores |

---

## Code Anti-Patterns (All Languages)

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Magic numbers / strings | Unreadable, breaks silently when value changes | Named constants or config values |
| Deeply nested conditionals (>3 levels) | Cognitive overload, hard to test | Early returns, guard clauses, extract functions |
| Functions doing >1 thing | Hard to test, hard to reuse, hard to name | Single Responsibility — one function, one job |
| Catching all exceptions silently | Hides bugs, masks data loss | Catch specific exceptions; log with context |
| Copy-pasted code blocks (>5 lines, 2x) | Change one, forget the other → divergence | Extract function or class; 3 copies = abstraction time |
| Comments explaining WHAT | Code already says what; comment rots as code changes | Name variables/functions well; comment only WHY |
| TODO comments committed | Permanent debt that never gets paid | File an issue; don't commit TODO |
| Hardcoded environment values | Breaks in different environments | Environment variables or config files |

---

## API / Integration Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| No retry logic on external calls | Single transient failure = user-visible error | Exponential backoff with jitter, max 3 retries |
| Ignoring HTTP status codes | `200 OK` with an error body is still a failure | Parse status code before parsing body |
| Synchronous calls inside loops | N API calls = N × latency | Batch where possible; parallel where independent |
| No timeout on external calls | One slow dependency hangs the entire request | Always set timeout; default 10s for user-facing, 60s for batch |
| Storing API responses without TTL | Stale data delivered as fresh | Cache with explicit expiry; invalidate on mutation |
| Secrets in API request logs | PII / credentials leak into log storage | Redact headers and body fields before logging |

---

## Database / Storage Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| SELECT * in production queries | Fetches unused columns, breaks if schema changes | Select only needed columns explicitly |
| No index on filter columns | Full table scan at scale | Index foreign keys and commonly filtered columns |
| Raw string interpolation in SQL | SQL injection (CRITICAL OWASP A03) | Parameterized queries always |
| Schema changes without migration | Breaks existing data | Write reversible migration; test on copy of prod data |
| Deleting data without soft-delete | Unrecoverable loss | Add `deleted_at` column; filter in queries |
| Transactions spanning multiple requests | Long-held locks, deadlocks | Keep transactions short; one DB round-trip |
| ORM N+1 queries | 1 query + N queries = silent performance disaster | Use eager loading / JOIN |

---

## LLM / Agent Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| No output validation after LLM call | Hallucinated structure breaks downstream code | Validate with Pydantic / schema before use |
| Prompt tuned to visible test cases | Brittle — fails on real inputs not in test set | Use SEALED holdout set (blind evaluation) |
| Retry loop without circuit breaker | Cost explosion on repeated failures | Max 3 retries then surface error to user |
| Logging full LLM response in prod | PII / sensitive data in log storage | Log metadata only: latency, tokens, confidence |
| Agent without threshold below which it stops | Low-confidence answer delivered as fact | Task Thresholds table: CRITICAL 0.98, STANDARD 0.90 |
| Overly broad agent (does everything) | Diluted KB, unfocused attention, hard to test | One agent = one domain = one KB section |
| Parallel agents sharing mutable state | Race conditions, non-deterministic output | Each agent produces isolated output; orchestrator aggregates |

---

## Security Anti-Patterns

> Full detail in `.claude/kb/security/` — this is the cross-domain summary.

| Anti-Pattern | Severity | Fix |
|--------------|----------|-----|
| Hardcoded API key / secret in source | CRITICAL | Environment variable or Secret Manager SDK |
| SQL / shell injection via string interpolation | CRITICAL | Parameterized queries; avoid shell=True |
| RLS disabled on Supabase table | CRITICAL | ENABLE ROW LEVEL SECURITY + CREATE POLICY |
| `roles/editor` or `roles/owner` on service account | CRITICAL | Least-privilege role specific to service |
| `NEXT_PUBLIC_` prefix on secret variable | CRITICAL | Remove prefix; access server-side only |
| `force_destroy = true` on production resource | CRITICAL | Remove or gate behind prod=false variable |
| console.log / print() with PII fields | HIGH | Log IDs only; redact email/token/password |
| Secrets accessed via os.environ in production | HIGH | Use Secret Manager SDK |

---

## Infrastructure Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Shared service account across functions | One compromise = all compromised | One SA per function with least-privilege |
| Manual infra changes in production | State drift, no rollback | All infra changes via Terraform / IaC |
| No DLQ on Pub/Sub subscription | Lost messages on processing failure | Configure Dead Letter Topic, max 5 retries |
| `terraform destroy` without confirmation | Irreversible data loss | Add `prevent_destroy` lifecycle; require explicit prompt |
| Secrets in Cloud Run `--set-env-vars` | Visible in GCP console to anyone with access | Use `--set-secrets` with Secret Manager reference |
| Auto-scaling without max instance limit | Cost explosion under attack or bug | Set `--max-instances` on every Cloud Run service |

---

## References

- Security detail: `.claude/kb/security/`
- OWASP checks: `.claude/kb/security/patterns/owasp-checks.md`
- GCP safety: `.claude/rules/gcp-safety.md`
- Python style: `.claude/rules/python-style.md`
- SDD rules: `.claude/rules/sdd-workflow.md`
