---
name: meeting
description: Analyze meeting notes, Slack threads, or email threads — extract decisions, requirements, action items, and blockers
---

# Meeting Command

> Delegates to the `meeting-analyst` agent for structured extraction from any communication format.

## Usage

```bash
/meeting notes/standup-2026-06-28.md      # Analyze a meeting notes file
/meeting "paste your notes here"           # Inline text
/meeting --output define                   # Extract directly into a DEFINE doc
```

---

## What It Does

1. Reads the provided notes/text (file or inline)
2. Invokes `meeting-analyst` agent
3. Extracts: decisions, requirements, action items, open questions, blockers, and implicit signals
4. Produces structured output in one of two modes:

| Mode | Output | When |
|------|--------|------|
| Default | Structured summary (decisions + AIs + blockers) | General meeting |
| `--output define` | Draft DEFINE document ready for `/define` | Requirements meeting |

---

## Agent Delegation

```markdown
Invoke: meeting-analyst
Input: {file path or inline text} + output mode flag
```

## See Also

- **Agent:** `.claude/agents/communication/meeting-analyst.md`
- **Next step (if --output define):** `/define .claude/sdd/features/DEFINE_{FEATURE}.md`
