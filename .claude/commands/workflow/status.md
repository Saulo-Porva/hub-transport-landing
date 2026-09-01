---
name: status
description: Generate a comprehensive project status report — active SDD features, git state, agent recommendations, and health assessment
---

# Status Command

Generates a project health report by scanning the SDD workspace, git history, and codebase indicators. Runs inline — no agent delegation.

## Usage

```bash
/status                    # Full project status report
/status "sprint review"    # Status with context label for the report header
```

---

## Execution Process

Execute all steps inline and produce the report directly.

### Step 1: Scan SDD Workspace

```text
# Active features (in progress)
Glob(".claude/sdd/features/BRAINSTORM_*.md")
Glob(".claude/sdd/features/DEFINE_*.md")
Glob(".claude/sdd/features/DESIGN_*.md")
Glob(".claude/sdd/reports/BUILD_REPORT_*.md")

# Recently shipped (archive)
Glob(".claude/sdd/archive/*/SHIPPED_*.md")  → last 5

# Security reports
Glob(".claude/sdd/reports/SECURITY_REPORT_*.md")
```

For each active document: read first 15 lines to extract feature name, status field, and date. Determine SDD phase. Flag if blocked (status contains "blocked" or "❌").

### Step 2: Check Git State

```bash
git log --oneline -10
git status --short
git branch --show-current
gh pr list --state open --limit 5 2>/dev/null || echo "no gh CLI"
```

### Step 3: Detect Project Health

```text
# Stack detection
Glob("**/package.json")          → Next.js / Node
Glob("**/pyproject.toml")        → Python
Glob("**/*.tf")                  → Terraform
Glob("supabase/migrations/**")   → Supabase
Glob("**/Dockerfile")            → Container

# Test coverage signal
Glob("**/test_*.py")
Glob("**/*.test.ts")
Glob("**/*.spec.ts")

# KB coverage
Glob(".claude/kb/*/index.md")    → available domains

# TODO debt
Grep("TODO|FIXME|HACK", output_mode="count")

# Docs presence
Glob("CLAUDE.md")
Glob("README.md")
```

Compare detected tech against available KB domains — flag gaps.

### Step 4: Recommendations

- **Next SDD step:** if feature is in Define → suggest `/design`; in Design → suggest `/build`; nothing active → suggest `/brainstorm`
- **Agent match:** based on detected stack (e.g., Next.js files → nextjs-specialist; Supabase migrations → supabase-specialist; .tf → infra-deployer)
- **Health action:** most urgent issue (uncommitted changes, stale branch, missing docs, high TODO count)

---

## Output Format

```markdown
# Project Status Report
**Project:** {name from package.json / pyproject.toml / dir name}
**Branch:** {current branch}
**Date:** {today}
{context label if provided}

---

## Active Work

| Feature | Phase | Status | Last Updated |
|---------|-------|--------|--------------|
| {name} | {Brainstorm/Define/Design/Build} | {In Progress / Blocked / Ready} | {date} |

> If empty: "No active SDD features. Use `/brainstorm` or `/define` to start."

## Recently Shipped

| Feature | Shipped |
|---------|---------|
| {name} | {date} |

> If empty: "No shipped features in archive."

## Recent Activity

| Hash | Message |
|------|---------|
| {hash} | {message} |

**Uncommitted:** {N} files | **Open PRs:** {N or N/A}

## Project Health

| Check | Status | Detail |
|-------|--------|--------|
| Tests | {Found/None} | {N} test files |
| Docs | {Present/Missing} | CLAUDE.md {✅/❌}, README {✅/❌} |
| Uncommitted | {Clean/N files} | {list if < 6} |
| TODOs/FIXMEs | {N found} | |
| Stack | {Detected} | {list: Next.js, Python, Terraform, Supabase…} |
| KB coverage | {Covered/Gaps} | {tech without matching KB domain} |
| Security reports | {N reports} | {latest: PASSED/BLOCKED} |

## Recommendations

1. **Next step:** {what to do based on active work}
2. **Best agent:** {which agent + why}
3. **Health action:** {most urgent issue}

## Suggested Commands

| Command | Reason |
|---------|--------|
| `/{cmd}` | {why now} |
| `/{cmd}` | {why now} |
```

---

## Best Practices

Run `/status` at the **start of every session** to regain context without re-reading every file manually. If you just shipped something, run `/status` to confirm the archive is clean and no active work was accidentally left behind.

**Performance:** inline only — Glob, Grep, Read, Bash. No agent delegation. Typical run: under 30 seconds.
