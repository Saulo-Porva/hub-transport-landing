# OWASP Checks — Patterns

> Check implementations for OWASP A01/A02/A03/A05/A09 across Next.js 15, Python/Cloud Run, Supabase, and GCP stacks.
> Loaded by `owasp-checker` and `infra-auditor` (A05 section).

---

## A01:2021 — Broken Access Control

### Next.js 15 (TypeScript/TSX)

```text
CHECK: Server Action without auth guard
Pattern: Functions in `app/actions/*.ts` that mutate data (INSERT, UPDATE, DELETE)
         without a session/auth check before the mutation.
Look for:
  - `export async function` in actions files
  - No call to `getUser()`, `auth()`, `createClient()`, or similar auth check
  - Supabase client used directly without RLS verification
Severity: CRITICAL
OWASP: A01:2021 – Broken Access Control
Fix: Add `const { data: { user } } = await supabase.auth.getUser(); if (!user) throw new Error("Unauthorized");` before any mutation.

CHECK: API route without auth
Pattern: `app/api/**/route.ts` with GET/POST handlers that return user data
         without checking the session.
Severity: CRITICAL

CHECK: Missing middleware protection
Pattern: Protected routes (e.g., `/admin/**`, `/motorista/**`) without middleware.ts
         that validates session before the route renders.
Look for: Absence of `middleware.ts` or `matcher` not covering protected paths.
Severity: HIGH
```

### Python / Cloud Run

```text
CHECK: Webhook handler without identity verification
Pattern: HTTP handler functions receiving POST requests that don't validate
         the caller identity (no HMAC check, no Bearer token verification).
Look for: `@app.route` or `def handle_` with `request.json` but no signature check.
Severity: HIGH

CHECK: User data access without identity check
Pattern: Firestore/BigQuery queries filtering by a user-supplied ID
         without first verifying the caller IS that user.
Severity: CRITICAL
```

### Supabase (SQL migrations)

```text
CHECK: RLS not enabled on table
Pattern in .sql files:
  - `CREATE TABLE` not followed by `ALTER TABLE ... ENABLE ROW LEVEL SECURITY`
  - `ALTER TABLE ... DISABLE ROW LEVEL SECURITY`
Severity: CRITICAL
Fix: Add `ALTER TABLE {table_name} ENABLE ROW LEVEL SECURITY;` after table creation.

CHECK: SELECT without RLS policy
Pattern: Table has RLS enabled but no policy for SELECT operation.
         Anonymous role can read all rows.
Severity: HIGH
```

---

## A02:2021 — Cryptographic Failures

### Next.js 15

```text
CHECK: Sensitive data in localStorage
Pattern: `localStorage.setItem` with keys like "token", "password", "secret", "user", "email"
Severity: HIGH
Fix: Store sensitive session data in httpOnly cookies, not localStorage.

CHECK: MD5/SHA1 for password hashing
Pattern: `createHash('md5')`, `createHash('sha1')` near password/auth context
Severity: CRITICAL
Fix: Use bcrypt or argon2 for password hashing.

CHECK: NEXT_PUBLIC_ prefix on secret value
Pattern: Variable name starting with `NEXT_PUBLIC_` that contains "KEY", "SECRET", "TOKEN", "PASSWORD"
Severity: CRITICAL
Reason: NEXT_PUBLIC_ variables are embedded in client bundle — visible to browser.
Fix: Remove NEXT_PUBLIC_ prefix; access server-side only.
```

### Python / Cloud Run

```text
CHECK: Weak password hashing
Pattern: `hashlib.md5(` or `hashlib.sha1(` near password/auth context
Severity: CRITICAL
Fix: Use `bcrypt.hashpw()` or `argon2-cffi` library.

CHECK: Secrets via os.environ in production code
Pattern: `os.environ.get("API_KEY")`, `os.environ["SECRET"]`, `os.getenv("PASSWORD")`
         in non-test Python files
Severity: HIGH
Fix: Use Google Secret Manager SDK: `client.access_secret_version(name=...)`.
OWASP: A02:2021 – Cryptographic Failures
```

### GCP

```text
CHECK: Secret in Cloud Run environment variable
Pattern: In Cloud Build YAML or deployment scripts:
  `--set-env-vars` containing key names like KEY, SECRET, TOKEN, PASSWORD
Severity: CRITICAL
Fix: Use `--set-secrets` with Secret Manager references instead.
```

---

## A03:2021 — Injection

### Python

```text
CHECK: SQL injection via f-string
Pattern: `f"SELECT` or `f"INSERT` or `f"UPDATE` or `f"DELETE` with a variable in the query
         e.g.: f"SELECT * FROM users WHERE id = {user_id}"
Severity: CRITICAL
Fix: Use parameterized queries: `cursor.execute("SELECT ... WHERE id = %s", (user_id,))`
OWASP: A03:2021 – Injection

CHECK: Shell injection
Pattern: `subprocess.run(..., shell=True)` with any variable interpolated in the command
         e.g.: `subprocess.run(f"convert {filename} output.png", shell=True)`
Severity: CRITICAL
Fix: Use list form without shell=True: `subprocess.run(["convert", filename, "output.png"])`

CHECK: Code injection
Pattern: `eval(user_input)`, `exec(user_input)`, `__import__(user_input)`
Severity: CRITICAL
Fix: Never eval user input. Use allowlists or structured data instead.
```

### Next.js 15

```text
CHECK: XSS via dangerouslySetInnerHTML
Pattern: `dangerouslySetInnerHTML={{ __html: ` with a variable that comes from user input
         or an API response that isn't sanitized.
Severity: CRITICAL
Fix: Use DOMPurify.sanitize() before setting innerHTML, or avoid dangerouslySetInnerHTML.

CHECK: Template literal injection in HTML context
Pattern: Template literals building HTML strings that include unsanitized user variables
         e.g.: `` `<div>${userInput}</div>` `` used in innerHTML assignment
Severity: HIGH
```

---

## A05:2021 — Security Misconfiguration

### Next.js 15

```text
CHECK: Missing Content-Security-Policy
Pattern: `next.config.js` or `next.config.ts` with `headers()` function but no
         Content-Security-Policy header defined.
Severity: MEDIUM
Fix: Add CSP header in next.config.ts headers() return.

CHECK: NEXT_PUBLIC_ on sensitive variable
Pattern: .env files or process.env references with NEXT_PUBLIC_ prefix on
         variable names containing KEY, SECRET, TOKEN, PASSWORD.
Severity: CRITICAL
(Same as A02 check — dual classification)

CHECK: Missing CORS configuration on API route
Pattern: API routes that don't set Access-Control-Allow-Origin header
         and accept cross-origin POST requests.
Severity: MEDIUM
```

### GCP

```text
CHECK: Cloud Run with --allow-unauthenticated on internal service
Pattern: In Cloud Build YAML or deploy scripts:
  `--allow-unauthenticated` on a service that should only be called by Pub/Sub or other internal services.
Severity: HIGH
Fix: Remove --allow-unauthenticated; use service accounts and IAM for internal invocation.

CHECK: IAM roles/editor or roles/owner on service account
Pattern in Terraform .tf or gcloud commands:
  `roles/editor` or `roles/owner` bound to a service account
Severity: CRITICAL
Fix: Use least-privilege roles specific to the service's needs.
```

### Terraform

```text
CHECK: force_destroy on critical resources
Pattern: `force_destroy = true` on `google_storage_bucket` or `google_sql_database_instance`
Severity: CRITICAL
Fix: Remove force_destroy or gate behind a variable that defaults to false in prod.

CHECK: deletion_protection disabled on database
Pattern: `deletion_protection = false` on Cloud SQL resources
Severity: HIGH
Fix: Set deletion_protection = true for production databases.

CHECK: prevent_destroy not set
Pattern: Critical resource blocks (databases, storage) without `lifecycle { prevent_destroy = true }`
Severity: MEDIUM
```

---

## A09:2021 — Security Logging Failures

### Next.js 15

```text
CHECK: console.log with PII fields
Pattern: `console.log(` followed by object/variable names containing:
  "email", "userId", "user_id", "password", "token", "phone", "address", "ssn", "dob"
Severity: HIGH
Fix: Remove PII from logs. Log IDs only: `console.log("User action", { userId: user.id.substring(0, 8) })`

CHECK: No logging on authentication failure
Pattern: Auth failure code paths (401 returns, password check failures) without any log call.
Severity: MEDIUM
Fix: Log authentication failures with timestamp, IP, and attempt count (no passwords).
```

### Python / Cloud Run

```text
CHECK: print() instead of structured logging
Pattern: `print(` in non-test Python files in Cloud Run functions
Severity: HIGH
Fix: Replace with `logging.getLogger(__name__).info(...)` with structured JSON output.
OWASP: A09:2021 – Security Logging Failures

CHECK: Logging full request body
Pattern: `logger.info(request.json())` or `logging.info(event)` where event is a full webhook payload
         (may contain PII or secrets passed in headers/body)
Severity: HIGH
Fix: Log only specific safe fields: `logger.info({"message_id": msg_id, "type": event_type})`

CHECK: Missing audit trail for sensitive operations
Pattern: Functions that delete users, change permissions, or process payments
         without a structured audit log entry.
Severity: MEDIUM
Fix: Add structured audit log: `logger.info({"audit": True, "action": "delete_user", "actor": actor_id, "target": target_id})`
```

---

## Supabase-Specific Checks (infra-auditor domain)

```text
CHECK: Table with RLS disabled
Pattern in SQL migrations:
  - `ALTER TABLE {name} DISABLE ROW LEVEL SECURITY`
  - `CREATE TABLE {name}` with no subsequent `ENABLE ROW LEVEL SECURITY` in same migration
Severity: CRITICAL
Fix: `ALTER TABLE {name} ENABLE ROW LEVEL SECURITY;`

CHECK: No policies defined after enabling RLS
Pattern: `ENABLE ROW LEVEL SECURITY` present but no `CREATE POLICY` for the table
Severity: HIGH
Fix: Add appropriate SELECT/INSERT/UPDATE/DELETE policies per role.
```
