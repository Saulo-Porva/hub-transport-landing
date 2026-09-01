# Secrets Patterns — Concepts

> Canonical secret detection patterns with false-positive exclusions. Loaded by `secrets-scanner`.

---

## Pattern Categories

### Category 1: API Keys

| Pattern | Regex | Severity | Notes |
|---------|-------|----------|-------|
| Stripe live key | `sk_live_[a-zA-Z0-9]{20,}` | CRITICAL | Live key in any file |
| Stripe test key | `sk_test_[a-zA-Z0-9]{20,}` | HIGH | Test key may reach prod |
| Google API key | `AIza[0-9A-Za-z\-_]{35}` | CRITICAL | GCP/Maps/Firebase keys |
| AWS Access Key ID | `AKIA[0-9A-Z]{16}` | CRITICAL | Always exploitable |
| GitHub token | `ghp_[a-zA-Z0-9]{36}` | CRITICAL | Personal access token |
| Slack token | `xox[baprs]-[0-9a-zA-Z\-]+` | CRITICAL | Bot/app/user tokens |
| Sendgrid key | `SG\.[a-zA-Z0-9\-_]{22}\.[a-zA-Z0-9\-_]{43}` | CRITICAL | API key |

### Category 2: JWT Tokens

| Pattern | Regex | Severity | Notes |
|---------|-------|----------|-------|
| Hardcoded JWT | `eyJ[A-Za-z0-9\-_]+\.eyJ[A-Za-z0-9\-_]+\.[A-Za-z0-9\-_]+` | CRITICAL | Embedded JWT is never safe |

Exception: JWT patterns in test fixtures at `*.spec.ts`, `*.test.ts` → HIGH (not CRITICAL).

### Category 3: Connection Strings

| Pattern | Regex | Severity | Notes |
|---------|-------|----------|-------|
| PostgreSQL DSN with credentials | `postgresql://[^:]+:[^@]+@[^/]+` | CRITICAL | Username:password embedded |
| MySQL DSN with credentials | `mysql://[^:]+:[^@]+@[^/]+` | CRITICAL | |
| MongoDB URI with credentials | `mongodb://[^:]+:[^@]+@[^/]+` | CRITICAL | |
| Redis URL with password | `redis://:([^@]+)@[^/]+` | CRITICAL | |

Exception: `localhost` in the host part → downgrade to MEDIUM (local dev pattern).

### Category 4: GCP Service Account Keys

| Pattern | What to detect | Severity |
|---------|---------------|----------|
| SA JSON key file | File containing `"type": "service_account"` | CRITICAL |
| Private key in file | `-----BEGIN RSA PRIVATE KEY-----` or `-----BEGIN EC PRIVATE KEY-----` or `-----BEGIN PRIVATE KEY-----` | CRITICAL |
| Private key in JSON field | `"private_key": "-----BEGIN` | CRITICAL |

### Category 5: Hardcoded Passwords

| Pattern | Regex | Severity | Notes |
|---------|-------|----------|-------|
| Password assignment | `(?i)password\s*[=:]\s*["'][^"']{8,}["']` | CRITICAL | Any variable assignment |
| Passwd variant | `(?i)passwd\s*[=:]\s*["'][^"']{8,}["']` | CRITICAL | |
| Secret assignment | `(?i)secret\s*[=:]\s*["'][^"']{8,}["']` | CRITICAL | |
| Token assignment | `(?i)token\s*[=:]\s*["'][^"']{8,}["']` | HIGH | May be legitimate variable name |

### Category 6: Generic Base64 Secrets

| Pattern | Condition | Severity |
|---------|-----------|----------|
| Long base64 string | `[A-Za-z0-9+/]{40,}={0,2}` AND context suggests it's a secret (var name contains "key", "secret", "token", "password") | MEDIUM |

---

## False-Positive Exclusion Logic

Apply these checks **before** reporting any finding. A finding that matches exclusion criteria is either downgraded (from CRITICAL → HIGH) or skipped entirely.

### File-Level Exclusions (downgrade CRITICAL → HIGH)

```text
If file path matches any of:
  - test_*.py
  - *.test.ts / *.test.js
  - *.spec.ts / *.spec.js
  - **/__tests__/**
  - **/fixtures/**
  - **/mocks/**
  - **/__mocks__/**
Then: All findings from this file → HIGH (not CRITICAL)
Reason: Test files legitimately contain fake secret patterns for testing
```

### Value-Level Exclusions (skip entirely)

```text
If the detected secret value matches any of:
  - "YOUR_API_KEY" / "your-api-key" / "YOUR_KEY_HERE"
  - "xxx" / "xxxx" / "XXXX..."
  - "placeholder" / "PLACEHOLDER"
  - "<redacted>" / "<YOUR_KEY>"
  - "example" / "EXAMPLE"
  - "changeme" / "CHANGE_ME"
  - "todo" / "TODO"
  - "test" / "TEST"
  - value length < 8 characters
Then: Skip — this is a documentation/template value, not a real secret
```

### Comment-Level Exclusions (skip entirely)

```text
If the line containing the finding also contains:
  # test / # fixture / # example / # placeholder / # fake / # dummy
  // test / // example / // placeholder / // fake
  /* example */ / /* placeholder */
Then: Skip — explicitly marked as non-secret
```

### Environment-Variable Name Exclusions (downgrade to MEDIUM)

```text
If the secret appears as a reference to an env var (not a hardcoded value):
  os.environ.get("API_KEY")           → Not a secret value, but flag as HIGH (infra-auditor handles this)
  process.env.SOME_VAR                → Not a secret value
  ${process.env.NEXT_PUBLIC_ANYTHING} → NEXT_PUBLIC_ prefix warrants check by owasp-checker
```

---

## Detection Strategy

The `secrets-scanner` should apply patterns in this order:

1. **Load exclusion lists** from this file before scanning
2. **Glob source files** (see `quick-reference.md` for patterns)
3. **For each file:**
   a. Check if file matches file-level exclusion (test/fixture paths)
   b. Read file content line by line
   c. Apply each regex pattern
   d. For each match: check value exclusions and comment exclusions
   e. Determine final severity (CRITICAL / HIGH / MEDIUM / skipped)
   f. Emit YAML finding if not skipped
4. **Assign IDs:** `SS-001`, `SS-002`, ... sequentially

---

## Fix Templates by Category

See `patterns/fix-suggestion-format.md` for the full format. Quick examples:

```text
# Hardcoded API key
current: API_KEY = "sk_live_abc123xyz..."
fix:     API_KEY = os.environ.get("API_KEY")  # or Secret Manager SDK
reason:  Hardcoded live API key in source code is a CRITICAL secret exposure.
         If committed, this key is visible in git history forever.

# JWT in code
current: const token = "eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyIjoiYWRtaW4ifQ..."
fix:     const token = getAuthToken(); // retrieve from secure session
reason:  Hardcoded JWT grants persistent access even after password changes.

# DB connection string
current: DATABASE_URL = "postgresql://admin:secret123@db.example.com/prod"
fix:     DATABASE_URL = os.environ.get("DATABASE_URL")  # set via Secret Manager
reason:  Credentials in connection string are exposed in source and git history.
```
