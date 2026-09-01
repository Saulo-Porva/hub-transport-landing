# Fix Suggestion Format — Patterns

> Standard format for `current_code / fix_code / reason` triples in scanner YAML output.
> All scanners must follow this format exactly for orchestrator aggregation to work.

---

## YAML Finding Structure

Every finding in the scanner YAML must have these fields populated:

```yaml
- id: "SS-001"              # Scanner prefix + sequential number
  severity: "CRITICAL"      # CRITICAL | HIGH | MEDIUM | LOW
  title: "Hardcoded Stripe API Key"
  file: "src/lib/payments.ts"
  line: 14
  current_code: |
    const apiKey = "sk_live_abc123xyz987def456...";
  fix_code: |
    const apiKey = process.env.STRIPE_SECRET_KEY;
    // Set STRIPE_SECRET_KEY in Vercel env vars → Secret Manager
  reason: |
    A live Stripe API key hardcoded in source code is exposed to anyone with
    repository read access and persists in git history even after removal.
    This key enables full payment operations on the live account.
  owasp_ref: "A02:2021 – Cryptographic Failures"
```

---

## Field Rules

### `id`
- Format: `{PREFIX}-{NNN}` where NNN is zero-padded to 3 digits
- Increment sequentially within a single scanner run
- Do not reuse IDs across scanner runs (orchestrator treats each run independently)

### `severity`
- Must be exactly one of: `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`
- Never use `INFO`, `WARNING`, `ERROR`, or other variants
- See `quick-reference.md` for severity decision rules

### `title`
- 4-8 words, title case
- Pattern: `{What was found} {In/On/Via} {Where}`
- Examples:
  - ✅ `Hardcoded API Key Detected`
  - ✅ `SQL Injection via f-string Template`
  - ✅ `RLS Disabled on Users Table`
  - ❌ `security issue found` (too vague)
  - ❌ `There is a problem with the authentication` (too long)

### `file`
- Always relative path from repository root
- Use forward slashes even on Windows
- Examples: `src/lib/client.py`, `supabase/migrations/001_init.sql`
- Never absolute paths

### `line`
- The line number where the issue is found (1-indexed)
- If the issue spans multiple lines, use the first line of the problematic block

### `current_code`
- Exact code as it appears in the file (verbatim excerpt)
- Include 1-2 lines of context if it helps clarify the issue
- Use YAML block scalar `|` for multi-line
- Never truncate with `...` — show the actual offending code

### `fix_code`
- A copy-paste-ready replacement for `current_code`
- Must be a complete drop-in replacement (same function signature, same variable names)
- May include a comment explaining where to configure the secure value
- For CRITICAL/HIGH findings: fix_code is **required**
- For MEDIUM/LOW findings: fix_code is **recommended** (provide if straightforward)

### `reason`
- 2-4 sentences explaining WHY this is a security issue
- Answer: What is the risk? Who is affected? What could an attacker do?
- Do NOT just restate what the code does — explain the security consequence
- For OWASP findings: mention the OWASP category briefly

### `owasp_ref`
- Use the standard OWASP 2021 format: `"A0X:2021 – Category Name"`
- Set to `null` if the finding is not an OWASP violation (e.g., pure secrets detection)
- Valid values: `"A01:2021 – Broken Access Control"`, `"A02:2021 – Cryptographic Failures"`, `"A03:2021 – Injection"`, `"A05:2021 – Security Misconfiguration"`, `"A09:2021 – Security Logging Failures"`

---

## Examples by Finding Type

### CRITICAL — Hardcoded Secret

```yaml
- id: "SS-001"
  severity: "CRITICAL"
  title: "Hardcoded Google API Key"
  file: "src/lib/maps.ts"
  line: 3
  current_code: |
    const MAPS_KEY = "AIzaSyAbCdEfGhIjKlMnOpQrStUv12345678";
  fix_code: |
    const MAPS_KEY = process.env.GOOGLE_MAPS_KEY;
    // Add GOOGLE_MAPS_KEY to Vercel environment variables
  reason: |
    A Google API key hardcoded in TypeScript source is visible to anyone with
    repository access and is permanently stored in git history. This key enables
    billing charges on the GCP project and may expose other Google services.
  owasp_ref: "A02:2021 – Cryptographic Failures"
```

### CRITICAL — SQL Injection

```yaml
- id: "OW-001"
  severity: "CRITICAL"
  title: "SQL Injection via f-string Query"
  file: "src/handlers/user.py"
  line: 28
  current_code: |
    query = f"SELECT * FROM users WHERE email = '{email}'"
    cursor.execute(query)
  fix_code: |
    query = "SELECT * FROM users WHERE email = %s"
    cursor.execute(query, (email,))
  reason: |
    Interpolating user-supplied input directly into a SQL query allows an attacker
    to manipulate the query structure. By injecting `' OR '1'='1` the attacker
    can bypass authentication or exfiltrate the entire database.
  owasp_ref: "A03:2021 – Injection"
```

### HIGH — RLS Table (no CRITICAL because it's a config gap, not active bypass)

```yaml
- id: "IA-001"
  severity: "CRITICAL"
  title: "RLS Disabled on payments Table"
  file: "supabase/migrations/005_payments.sql"
  line: 12
  current_code: |
    CREATE TABLE payments (
      id uuid PRIMARY KEY,
      user_id uuid REFERENCES auth.users(id),
      amount numeric NOT NULL
    );
    ALTER TABLE payments DISABLE ROW LEVEL SECURITY;
  fix_code: |
    CREATE TABLE payments (
      id uuid PRIMARY KEY,
      user_id uuid REFERENCES auth.users(id),
      amount numeric NOT NULL
    );
    ALTER TABLE payments ENABLE ROW LEVEL SECURITY;
    CREATE POLICY "users_own_payments" ON payments
      FOR ALL USING (user_id = auth.uid());
  reason: |
    Disabling Row Level Security on the payments table allows any authenticated
    user to SELECT, INSERT, UPDATE, or DELETE any payment record. An attacker
    with a valid session can read all financial data or modify other users' payments.
  owasp_ref: "A01:2021 – Broken Access Control"
```

### HIGH — console.log PII

```yaml
- id: "OW-002"
  severity: "HIGH"
  title: "PII Logged via console.log"
  file: "src/app/actions/auth.ts"
  line: 41
  current_code: |
    console.log("Login attempt:", { email, userId, ipAddress });
  fix_code: |
    console.log("Login attempt:", { userId: userId?.substring(0, 8), success: false });
  reason: |
    Logging user email and IP address exposes PII in log storage systems. Under
    GDPR/LGPD, log PII must be minimized and protected. Email addresses in logs
    are also useful to attackers enumerating valid accounts.
  owasp_ref: "A09:2021 – Security Logging Failures"
```

### MEDIUM — Missing CSP

```yaml
- id: "OW-003"
  severity: "MEDIUM"
  title: "Missing Content-Security-Policy Header"
  file: "next.config.ts"
  line: 8
  current_code: |
    async headers() {
      return [{ source: "/(.*)", headers: [{ key: "X-Frame-Options", value: "DENY" }] }];
    }
  fix_code: |
    async headers() {
      return [{
        source: "/(.*)",
        headers: [
          { key: "X-Frame-Options", value: "DENY" },
          { key: "Content-Security-Policy", value: "default-src 'self'; script-src 'self' 'unsafe-inline'" },
          { key: "X-Content-Type-Options", value: "nosniff" },
        ],
      }];
    }
  reason: |
    Without a Content-Security-Policy header, browsers allow scripts from any
    origin to execute, enabling XSS attacks to load external malicious scripts.
    A CSP restricts script sources to trusted domains.
  owasp_ref: "A05:2021 – Security Misconfiguration"
```

### LOW — Missing audit trail

```yaml
- id: "OW-004"
  severity: "LOW"
  title: "No Audit Log on Sensitive Operation"
  file: "src/app/actions/admin.ts"
  line: 67
  current_code: |
    await supabase.from("users").delete().eq("id", userId);
    return { success: true };
  fix_code: |
    await supabase.from("users").delete().eq("id", userId);
    console.log(JSON.stringify({ audit: true, action: "delete_user", actor: adminId, target: userId, ts: new Date().toISOString() }));
    return { success: true };
  reason: |
    Deleting a user without an audit log makes it impossible to investigate
    unauthorized deletions or track admin actions for compliance purposes.
  owasp_ref: "A09:2021 – Security Logging Failures"
```

---

## Quality Rules for Fix Suggestions

1. **Fix must compile** — the fix_code must be syntactically valid in the target language
2. **Fix must preserve behavior** — the fixed version should do what the original intended, just safely
3. **Fix must be actionable** — if the fix requires external configuration (env vars, Secret Manager), include a comment showing what to configure
4. **Fix must not introduce new issues** — don't swap SQL injection for shell injection
5. **Reason must explain impact** — not just "this is bad" but "an attacker can do X because of this"
