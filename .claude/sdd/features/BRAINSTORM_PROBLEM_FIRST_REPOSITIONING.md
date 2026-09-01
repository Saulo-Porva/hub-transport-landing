# BRAINSTORM: Problem-First Repositioning (v3 of the site)

> Exploratory session to clarify intent and approach before requirements capture

## Metadata

| Attribute | Value |
|-----------|-------|
| **Feature** | Problem-First Repositioning |
| **Date** | 2026-09-01 |
| **Author** | brainstorm-agent (with the user's own detailed creative brief as primary input) |
| **Status** | Approaches Identified — key forks pending user confirmation |

---

## Initial Idea

**Raw Input:** The user provided a full, highly detailed repositioning brief (verbatim, ~2000 words) arguing the site should stop selling "7 software products" and instead sell one competency — *"You show us where your operation loses time, money or control. We turn it into software."* — with every case study restructured as **Problem → System → Result**, a new problem-first hero, a "what's slowing you down" selector, three cases promoted to flagship status (Fleet Control, Workforce, AI Real Estate), the rest listed as "more systems," a founder-led studio identity (not hidden, not corporate), and a strong image discipline (70% real product, 20% diagrams, 10% photography — no stock imagery).

**Context Gathered:**
- The current site (built earlier this session) already carries the seed of this idea: hero says *"I build the software your business is still missing,"* there's an Understand → Design → Solve track, and 7 case pages exist with Problem → Approach → Solution → Result structure and real/recreated screen mockups per case.
- What's missing per the user: the hero still doesn't open with the *reader's* problem (it opens with "I build..."), all 7 cases are presented as equals with no featured tier, case copy is still fairly prose-heavy rather than before/after/result blocks, and the demo form at the bottom still says "Fleet name," "Fleet size" — a TruckPilot-specific artifact that contradicts the broadened positioning.
- Product naming clarified in the brief: **"Rapportini" is proposed to be renamed "Fleet Control"** for the external-facing site — same product (trip tracking, fuel/workshop cost, one-click trailer load view, dispatch board, 60-day → same-month invoicing), better external name. **TruckPilot stays TruckPilot** (name is fixed — already on business cards) and keeps its own distinct WhatsApp-compliance-tools story, now framed as a separate, narrower case from Fleet Control.
- The brief flags that the whole site currently lives on `truckpilot.netlify.app` — a domain mismatch with the new "SHS solves any operational problem" positioning. Raised as a question, not a decision.

**Technical Context Observed (for Define):**

| Aspect | Observation | Implication |
|--------|-------------|-------------|
| Likely Location | Root of repo — `index.html` + 7 `case-*.html` files + `styles.css` + `icons/` | No build step; every change is a direct HTML/CSS/JS edit across up to 8 files |
| Relevant KB Domains | `marketing-copy` (case-study structure, headline formulas, social proof), `ui-design`, `ux` | `marketing-copy/patterns/case-study-structure.md` needs updating — this brief adds a "Before" sub-step and a "Technology" sub-step the current pattern doesn't have |
| Existing agents | `product-quality-auditor` orchestrates `ui-designer` + `ux-specialist` + `marketing-specialist` + `landing-parity-auditor` for landing/case pages | `landing-parity-auditor`'s rule already explicitly allows a declared "flagship" tier with a link to why — the 3-featured/4-more split is compatible with the existing rule, not a violation of it |
| IaC Patterns | N/A — static site on Netlify | Domain separation (see open question) is a hosting/DNS decision, not a code change |

---

## Discovery Questions & Answers

| # | Question | Answer | Impact |
|---|----------|--------|--------|
| 1 | Visual theme — keep dark, go light, or hybrid? | **Light/editorial** — white/off-white base, giant typography, floating result cards, real screenshots. Dark sections allowed locally, not the base. | Full re-theme of all 8 pages, `styles.css`, and every case page's card/panel colors |
| 2 | Is "Fleet Control" a full rename of Rapportini (file, nav, links) or just a display-name change? | **Full rename** — file, URL, nav, all internal links move from `case-rapportini.html` to `case-fleet-control.html`; "Rapportini" no longer appears on the public site | Requires a file rename + a repo-wide link sweep (index.html cards, hero constellation, nav) |
| 3 | Should TruckPilot move to its own domain/site now, or stay in this repo with just the demo form genericized? | **Stay in this repo for now** — only genericize the demo form (drop "Fleet name"/"Fleet size" as required, accept any problem type). Domain split explicitly deferred, not executed today. | Small, safe content fix; no infra/DNS work this pass |
| 4 | Which tagline? | **"Show us the bottleneck. We'll build the system."** | New hero headline; replaces "I build the software your business is still missing." everywhere it's echoed (meta description, etc.) |

**Minimum Questions:** 3 ✅ (4 asked)

---

## Approaches Explored

### Approach A: Full rebuild in one pass ⭐ Not recommended as a single step

**Description:** Rewrite hero, add problem-selector, restructure all 7 case pages to the new Problem/Before/System/Result/How/Product/Impact/Technology/CTA template, rename Rapportini → Fleet Control, add floating result cards, resolve theme, all in one long build session.

**Pros:**
- Fastest path to the finished vision the user described.

**Cons:**
- Several genuine forks are still open (theme, rename scope, domain). Building before they're resolved risks redoing large amounts of work if the answer is "no, the other way" — which is exactly what happened earlier this session when a big visual pass was built before the direction was confirmed.
- The user's own closing line offers to design the homepage section-by-section first, before it becomes a build prompt — a sequencing they explicitly proposed.

---

### Approach B: Resolve the forks, then rebuild top-down (home first, then cases) ⭐ Recommended

**Description:** Answer the 4 blocking questions above, then rebuild in a fixed order: (1) hero + problem-selector + results band on `index.html`, (2) case-study template evolution (Before/Technology sub-steps, floating result cards) applied first to the 3 featured cases, (3) roll the same template to the 4 "more systems" cases, (4) fix the demo form copy, (5) Fleet Control rename executed as its own focused step.

**Pros:**
- Matches the user's own suggested sequencing (design first, confirm, then build).
- Home page sets the pattern (headline formulas, floating-card style, spacing) that every case page then reuses — building it first avoids inconsistency between cases built early vs. late.
- Each step is independently shippable and reviewable, consistent with how the rest of this session has gone (build → show → confirm → push).

**Cons:**
- Slower than Approach A — more checkpoints.

---

### Approach C: Ship copy/positioning changes first, defer visual overhaul

**Description:** Rewrite all headlines, problem-selector, Before/Result copy, and the demo form using the *current* dark visual system (no theme change), then revisit visual direction (light/hybrid, floating cards) as a second pass once the copy is validated.

**Pros:**
- De-risks the biggest open question (theme) by not blocking copy work on it.
- Content-only changes are fast to review.

**Cons:**
- The floating-result-card treatment and "giant typography + white space" feel are central to how the user described the new case-study pages actually *selling* — shipping copy without the visual language means the first review won't show the real intended effect, likely triggering another round of "not what I meant."

---

## Selected Approach

| Attribute | Value |
|-----------|-------|
| **Chosen** | Approach B |
| **User Confirmation** | Pending — this document is presented alongside the chat questions in the same turn |
| **Reasoning** | Matches the user's own proposed sequencing; every prior large visual pivot this session that skipped an explicit confirmation step needed a redo — Approach B is the one that doesn't repeat that pattern at 10x the current scope |

---

## Key Decisions Made (from the user's brief — not open questions)

| # | Decision | Rationale | Alternative Rejected |
|---|----------|-----------|----------------------|
| 1 | Site sells one competency ("turn operational problems into software"), not 7 products | Buyers arrive with a pain, not a product name in mind | Product-first catalog framing (current v2) |
| 2 | Positioning: founder-led software studio (not hidden freelancer, not fake multinational) | Builds trust now, allows scaling the team later without a rebrand | Anonymous "software house" framing |
| 3 | 3 cases promoted to featured tier (Fleet Control, Workforce, AI Real Estate); 4 listed as "more systems we've built" (TruckPilot, Contabile, Health Anywhere, Workout) | Avoids an endless homepage; still honest since it's a declared tier, not accidental imbalance | All 7 equal-weight (v2's approach) |
| 4 | Case-study template gains two sub-steps: **Before** (bullet list of the old broken state) and **Technology** (short, discreet stack mention near the end) | Before/After framing sells faster than prose; technology belongs late, after the value case is made, and stays light | Keep the current 4-section (Problem/Approach/Solution/Result) template unchanged |
| 5 | New home section: "Don't see your exact problem here?" — explicit product/customization/new-project funnel | Directly supports the "we solve problems, not just sell 7 SKUs" positioning | Leaving the generic "Tell me about it" CTA as the only funnel |
| 6 | Image discipline: ~70% real product (screenshots/mockups), 20% diagrams/process, 10% photography; no stock imagery | Matches what's already been done this session (recreated real screens, no stock photos) — brief just formalizes it | Traditional corporate stock photography |
| 7 | Floating "result" mini-cards overlaid near hero/product images (e.g. "CMR received ✓", "Fuel: 312L", "Overtime ↓") | Cheap, high-impact way to show outcome without reading prose | Plain screenshots with no annotation |

---

## Features Removed (YAGNI) — none yet

Nothing has been explicitly cut at this stage; the brief is being taken close to as-given. YAGNI trimming, if any, will surface once the home page copy is drafted section-by-section (next step).

---

## Incremental Validations

| Section | Presented | User Feedback | Adjusted? |
|---------|-----------|---------------|-----------|
| Overall repositioning direction | ✅ (this doc + chat summary) | Confirmed via 4 answers below | — |
| 4 blocking-question answers | ✅ (chat, `AskUserQuestion`) | Light/editorial theme, full Fleet Control rename, form-only fix for TruckPilot domain, tagline confirmed | — |
| Home page rebuild (hero, problem selector, results band, about, track, 3+4 solutions tier, funnel, form, footer) — full light theme | ✅ (built + Chrome-tested on `index.html`, screenshots reviewed) | Pending user review | Fixed one i18n bug found during testing (IT/DE results band mixing languages) |
| Fleet Control case page (renamed from Rapportini, light theme, Before/Technology sub-steps added) | ✅ (built + Chrome-tested on `case-fleet-control.html`) | Pending user review | — |

**Minimum Validations:** 2 required — met. Home page and the first case-page template evolution are built and self-tested; still pending is the user's own review before the same treatment rolls out to the remaining 6 case pages.

---

## Suggested Requirements for /define

### Problem Statement (Draft)
The site currently sells "7 software products" of equal weight with a founder-first hero; it needs to sell one competency (turning operational bottlenecks into software) with a problem-first hero, a declared 3-featured/4-more case tier, and a Before/System/Result case template — while resolving the TruckPilot-domain and visual-theme questions that block full execution.

### Target Users (Draft)
| User | Pain Point |
|------|------------|
| Prospective client landing on the home page | Doesn't think in product names — thinks in operational pain ("too much WhatsApp," "15 days to invoice") |
| Saulo (site owner) | Needs the site's opening promise and its closing form to agree with each other — currently hero says "any business," form says "fleet name" |

### Success Criteria (Draft)
- [ ] Hero opens with the reader's problem, not the builder's pitch
- [ ] Home page has a working problem-selector section
- [ ] 3 cases read as featured, 4 as "more systems," both tiers still linking to full case pages
- [ ] Every case page follows Problem → Before → System → How it works → Product → Impact → Technology → CTA
- [ ] Rapportini fully repositioned as "Fleet Control" (scope depends on Q2 answer)
- [ ] Demo form no longer TruckPilot-specific
- [ ] Visual theme resolved and applied consistently across all 8 pages

### Constraints Identified
- TruckPilot's literal name cannot change (business cards) — only reframed as one case among featured ones, same constraint as v2
- No new privacy exposure — any new product imagery follows the same anonymization discipline already established this session
- No fabricated metrics — "15+ years" and any other founder-bio claim must be confirmed by the user, not assumed

### Out of Scope (Confirmed, for this pass)
- Actually provisioning a second Netlify site/domain for TruckPilot (infra decision, flagged not executed)
- Rebuilding Fast Health / Workforce Management / Real Estate's *actual product apps* — this is landing-page/case-study work only

---

## Session Summary

| Metric | Value |
|--------|-------|
| Questions Asked | 4 (in chat, this turn) |
| Approaches Explored | 3 |
| Features Removed (YAGNI) | 0 so far |
| Validations Completed | 0 of 2 (pending user response) |

---

## Next Step

**Status:** Build complete. All 8 pages (`index.html` + 7 case pages) rebuilt in the light/editorial theme, `case-rapportini.html` renamed to `case-fleet-control.html`, every case page evolved to Problem → Before → System → How It Works → Result → Technology → CTA, demo form genericized, open funnel section added. Self-tested in Chrome across all pages (EN + spot-checked IT/DE on the home page) — one i18n bug found and fixed (results band mixing languages on IT/DE).

**Ready for:** User review directly on the deployed site after push. Out of scope this pass, per the confirmed decisions: TruckPilot domain/hosting separation (deferred), and any further copy refinement the user requests after seeing it live.
