# Pattern: Landing Page Hierarchy

> The order information appears on a single long page determines what a skimming visitor actually
> absorbs. This is separate from copy quality — even great copy fails if it's placed after the
> point where most visitors stop scrolling.

---

## Order (for this site's home page)

```
1. Hero          — who this is, one-line proof of range (rotating problem→solution ticker)
2. About         — the founder, briefly, establishing the studio positioning
3. How it works  — the repeatable process (Understand → Design → Solve), sets expectation
4. Solutions     — the proof: 7 equal-weight case cards
5. Contact       — the ask
```

Each section earns the right to the next: Hero earns attention, About earns trust in the person,
How-it-works earns trust in the process, Solutions delivers the proof, Contact is the only place
that asks for something.

---

## Never Ask Before Proving

The contact form / demo request never appears before the Solutions section on the home page. Asking
for a visitor's information before showing evidence inverts the trust sequence — it reads as a lead
capture front-loaded ahead of substance, which contradicts the studio positioning
(see `concepts/positioning-statement.md`).

---

## Equal-Weight Enforcement

The Solutions section is the one place where hierarchy discipline and the parity rule intersect:
every case card must carry the same visual weight (see
`.claude/agents/frontend/landing-parity-auditor.md`). A flagship case may get a declared badge and
a link to deeper content, but never a structurally bigger card by default.

---

## Case-Study Pages Don't Repeat This Hierarchy

Individual case pages follow `patterns/case-study-structure.md` instead — they're arrived at
already convinced enough to read one case in depth, so the hierarchy question there is about pacing
within Problem → Approach → Solution → Result, not about earning attention from zero.
