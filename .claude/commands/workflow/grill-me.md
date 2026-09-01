# Grill-Me Command

> Stress-test a DEFINE or DESIGN document through relentless interrogation before advancing to the next phase.

## Usage

```bash
/grill-me .claude/sdd/features/DEFINE_{FEATURE}.md
/grill-me .claude/sdd/features/DESIGN_{FEATURE}.md
```

## When to Run

| After | Before | Purpose |
|-------|--------|---------|
| `/define` | `/design` | Validate every requirement before investing in architecture |
| `/design` | `/build` | Validate every decision before investing in implementation |

---

## What This Does

Interrogates every aspect of the target document until each branch of the design tree is fully resolved. For each gap, ambiguity, or unvalidated assumption found:

1. Explores the codebase first — if the answer is derivable from existing code or patterns, derive it and confirm rather than ask cold.
2. Proposes a recommended answer.
3. Asks ONE question at a time, waiting for the user to respond before continuing.
4. Updates the document in place immediately after each confirmed answer.

Never accumulates answers and batch-updates — documents stay live throughout the session.

---

## Process

### Step 1: Load and Detect

```markdown
Read(<target-file>)
Read(.claude/sdd/templates/DEFINE_TEMPLATE.md) or Read(.claude/sdd/templates/DESIGN_TEMPLATE.md)
Read(.claude/CLAUDE.md)
```

Detect document type from filename (`DEFINE_*` vs `DESIGN_*`) and adapt the question tree accordingly.

### Step 2: Build Question Tree

Walk every section of the document and flag issues.

**For DEFINE documents — interrogate:**
- Problem Statement: specific enough to write an acceptance test for?
- Success Criteria: does every criterion have a measurable threshold (number, %, time)?
- Assumptions: are any unvalidated with no stated fallback?
- Open Questions: are there 🔴 Blocker items still `Open`?
- Out of Scope: does it explicitly exclude the most tempting adjacent features?
- Goals: are any MUST items actually SHOULD in disguise?
- Acceptance Tests: does every Scenario have a Given/When/Then that a tester can execute without asking questions?

**For DESIGN documents — interrogate:**
- Architecture: does every component have a single clear responsibility?
- Key Decisions: does each decision include at least one rejected alternative?
- File Manifest: does every DEFINE requirement trace to at least one file?
- Agent Assignments: are there files without a matched agent?
- Testing Strategy: does every Acceptance Test from DEFINE map to a test case?
- Error Handling: is every external call covered?
- Security Considerations: are auth, data isolation, and secret handling stated?

### Step 3: Interrogate — One Question at a Time

For each issue, use this format:

```
🔍 [Section] — [What is unclear or missing]

Recommended: [Concrete proposal based on codebase or context]

Correct? Or would you adjust?
```

Wait for the user's response before moving to the next issue.

### Step 4: Update Document Immediately

After each confirmed answer:

```markdown
Edit: <target-file>
  - Apply the answer to the relevant section
  - Preserve all other content
```

Never rewrite the whole document — apply surgical edits per answer.

### Step 5: Grill Summary

After all questions are resolved, append to the document:

```markdown
---

## Grill Summary

| Date | Questions | Sections Updated | Status |
|------|-----------|------------------|--------|
| {YYYY-MM-DD} | {N} | {comma-separated list} | ✅ Grilled — ready for {/design or /build} |
```

---

## Quality Gate

Document is "grilled" when all of the following are true:

**DEFINE:**
- [ ] No vague language without a measurable qualifier
- [ ] All 🔴 Blocker open questions answered
- [ ] All unvalidated Assumptions have a stated fallback
- [ ] Clarity Score verified at ≥ 12/15

**DESIGN:**
- [ ] Every file in the manifest has an agent assigned
- [ ] Every Key Decision has an alternative rejected
- [ ] Every DEFINE Acceptance Test maps to a test case
- [ ] Error handling covers every external dependency

---

## References

- Runs on: `.claude/sdd/features/DEFINE_{FEATURE}.md` or `DESIGN_{FEATURE}.md`
- Templates: `.claude/sdd/templates/`
- Phase flow: `/define` → `/grill-me DEFINE` → `/design` → `/grill-me DESIGN` → `/build`
