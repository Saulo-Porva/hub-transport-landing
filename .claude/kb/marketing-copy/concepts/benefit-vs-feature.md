# Benefit vs. Feature

> A feature is what the system does. A benefit is what changes for the person because of it. Copy
> that lists features without benefits reads like a spec sheet, not a reason to care.

---

## The Pattern

```
Feature: "Generates a timestamped PDF report automatically."
Benefit: "Ready to hand an inspector on the spot — nothing to assemble under pressure."
```

The benefit answers "so what?" from the reader's specific situation, not a generic one.

---

## How to Convert

1. State the feature plainly.
2. Ask "so what does that change for this specific person, in this specific moment?"
3. If the answer is generic ("saves time"), ask again — whose time, doing what, instead of what.
4. Write the benefit using the concrete moment, not the abstraction.

```
Feature → generic benefit (weak) → specific benefit (strong)
"Reads clocked hours automatically"
  → "saves time" (weak — saves time on what?)
  → "no manual timesheet entry" (better — still a feature restated)
  → "the manager stops rebuilding the roster from memory every Monday" (strong — names the moment)
```

---

## When to Keep the Feature Visible

Not every feature needs a benefit sentence attached — in a scannable list (like a case-study
"Solution" section), a feature phrased with implicit benefit is fine, as long as the surrounding
Problem/Result sections already established why it matters. Don't force a benefit clause onto every
bullet if it's already been earned by the section above it.

---

## Anti-Pattern: Benefit Without Feature

The opposite failure is just as common — a benefit claim with nothing concrete behind it
("dramatically improves efficiency"). If there's no feature that produces the claimed benefit,
the claim doesn't belong in the copy. See `concepts/social-proof.md` for the verifiability test.
