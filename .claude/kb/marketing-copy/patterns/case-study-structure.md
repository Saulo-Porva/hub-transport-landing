# Pattern: Case Study Structure

> The structure every case-study page on this site follows. Consistency here matters more than any
> single page's cleverness — a visitor who reads two cases should recognize the pattern instantly.

---

## Structure

```
1. Nav: logo → home, "← All solutions", primary CTA
2. Hero
   - Kicker: "Case Study 0N — {category}"
   - Headline: Formula 1/2/3 from headline-formulas.md
   - Subhead: one sentence, plain language, no jargon
   - Meta row: Client / Sector / Stack / Status
3. Stat band (optional): one hero number if a strong one exists, otherwise skip — don't force one
4. 01 / The Problem
   - 2-4 sentences, names who feels it and why it's urgent/costly
5. 02 / The Approach
   - 2-4 sentences, describes the actual decision made — not a generic process description
6. 03 / The Solution
   - Concrete feature list or deep-dive, benefit-attached (see benefit-vs-feature.md)
7. 04 / The Result
   - Restates the transformation, ends on the one verifiable metric/outcome chip
8. Closing CTA: two options — contact, and back to all solutions
9. Footer: same across all case pages
```

---

## Section Length Discipline

Each numbered section (01-04) should be readable in under 20 seconds for a skimming visitor. If a
section needs a full paragraph plus a feature grid plus a callout box, it has grown past what a
lightweight case needs — that depth belongs in a flagship case only, and only when explicitly
declared as one (see `.claude/agents/frontend/landing-parity-auditor.md` for the parity rule this
protects).

---

## Reused Copy Across Sections

Never let the Problem and Result sections restate the same sentence with different wording — Result
should reference the *transformation*, not repeat the *pain*. If a draft's Result section reads like
a rewritten Problem section, that's a sign the actual outcome hasn't been articulated yet.

---

## What NOT to Add

- A generic "why choose us" section — the case itself is the argument
- A pricing section on individual case pages — pricing conversations happen through the CTA, not
  pre-answered here
- Testimonial placeholders (see `concepts/social-proof.md`)
