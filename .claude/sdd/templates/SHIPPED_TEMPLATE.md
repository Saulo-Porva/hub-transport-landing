# SHIPPED: {Feature Name}

> Feature shipped on {YYYY-MM-DD}

## Metadata

| Attribute | Value |
|-----------|-------|
| **Feature** | {FEATURE_NAME} |
| **Ship Date** | {YYYY-MM-DD} |
| **Author** | ship-agent |

---

## Summary

{One paragraph describing what was built and the business value delivered.}

---

## Timeline

| Milestone | Date | Duration |
|-----------|------|----------|
| Define Started | {YYYY-MM-DD} | - |
| Define Complete | {YYYY-MM-DD} | {X days} |
| Design Complete | {YYYY-MM-DD} | {X days} |
| Build Complete | {YYYY-MM-DD} | {X days} |
| **Shipped** | {YYYY-MM-DD} | **Total: {X days}** |

---

## Metrics

| Metric | Value |
|--------|-------|
| **Total Files Created** | {N} |
| **Lines of Code** | {N} |
| **Tests Written** | {N} |
| **Test Coverage** | {X}% |
| **Build Iterations** | {N} |
| **Design Decisions** | {N} |

---

## What Was Built

### Components

| Component | Description |
|-----------|-------------|
| {Component 1} | {What it does} |
| {Component 2} | {What it does} |

### Files

| File | Purpose |
|------|---------|
| `{path/to/file1}` | {Purpose} |
| `{path/to/file2}` | {Purpose} |

---

## Success Criteria Verification

| Criterion | Target | Actual | Status |
|-----------|--------|--------|--------|
| {From DEFINE} | {Target} | {Actual} | ✅ / ❌ |
| {From DEFINE} | {Target} | {Actual} | ✅ / ❌ |
| {From DEFINE} | {Target} | {Actual} | ✅ / ❌ |

---

## Shadow Score (Sealed Tests)

> ship-agent evaluates `SEALED_{FEATURE}.md` scenarios against the shipped code.
> Builder never saw these scenarios — a passing shadow score means genuine requirement satisfaction.

| Scenario | Weight | Steps Passed | Status |
|----------|--------|--------------|--------|
| S-001 | HIGH | {X/Y} | ✅ / ❌ |
| S-002 | HIGH | {X/Y} | ✅ / ❌ |
| S-003 | MEDIUM | {X/Y} | ✅ / ❌ |
| S-004 | MEDIUM | {X/Y} | ✅ / ❌ |

```
shadow_score = (HIGH_passed/HIGH_total * 0.60) + (MEDIUM_passed/MEDIUM_total * 0.30) + (LOW_passed/LOW_total * 0.10)
```

| Score | Threshold | Status |
|-------|-----------|--------|
| **{X%}** | ≥ 90% | ✅ Passed / ❌ Failed |

> If shadow_score < 90%: open `/iterate` to address gaps before marking as shipped.

---

## Patterns Extracted

> ship-agent: identify patterns discovered during this build that are novel and reusable.
> For each novel pattern, add a file to `.claude/kb/{domain}/patterns/`.

### Pattern: {Pattern Name}

**When to use:** {Exact context — what problem this solves, when to reach for it}

**Domain:** {Which KB domain — agents / gcp / pydantic / etc.}

**KB Status:** ✅ Added to `.claude/kb/{domain}/patterns/{name}.md` / ❌ Already exists

```{language}
{Copy-paste ready code snippet demonstrating the pattern}
```

---

> Add `### Pattern:` for each novel reusable pattern discovered.
> If no novel patterns: write "No new patterns — existing KB covered this build."

---

## Lessons Learned

### Process

{What would you do differently in the process next time?}

- {Lesson 1: Be specific and actionable}
- {Lesson 2: Include what worked AND what didn't}

### Technical

{What technical insights were gained?}

- {Lesson 1: Technical discovery or pattern that worked}
- {Lesson 2: Technical challenge and how it was solved}

### Communication

{Where did early clarification help or would have helped?}

- {Lesson 1: What clarification prevented rework}
- {Lesson 2: What confusion could have been avoided}

### Tools & Libraries

{What tools or libraries proved valuable?}

- {Tool/Library 1: Why it was helpful}
- {Tool/Library 2: Why it was helpful}

---

## Recommendations for Future Work

| Area | Recommendation |
|------|----------------|
| {Area 1} | {Specific recommendation} |
| {Area 2} | {Specific recommendation} |

---

## Archived Artifacts

| Artifact | Location |
|----------|----------|
| BRAINSTORM | `./BRAINSTORM_{FEATURE}.md` (if Phase 0 was used) |
| DEFINE | `./DEFINE_{FEATURE}.md` |
| DESIGN | `./DESIGN_{FEATURE}.md` |
| BUILD_REPORT | `./BUILD_REPORT_{FEATURE}.md` |
| SHIPPED | `./SHIPPED_{DATE}.md` (this file) |

---

## Acknowledgments

{Optional: Note any particular challenges overcome or valuable contributions.}

---

*Feature archived on {YYYY-MM-DD} by ship-agent*
