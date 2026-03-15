# Design Review Rubric

Use this to score a captured screen and convert scores into ranked, actionable findings. Score each dimension 1–5, note evidence (measured values, not impressions), then rank fixes by impact. Hierarchy, typography, and color/contrast usually dominate perceived quality — weight them accordingly.

## Scoring sheet

| # | Dimension | 1 — broken | 3 — passable | 5 — excellent | Common evidence to capture |
| --- | --- | --- | --- | --- | --- |
| 1 | **Hierarchy** | No focal point; everything same weight/size; eye wanders | Primary action findable but competes | One clear focus; eye lands as intended; order from size/weight/space | First-second eye path; H1 vs body size/weight delta; count of "shouting" elements |
| 2 | **Typography** | Random sizes, ≥4 weights, body too small, bad measure | Mostly consistent; minor measure/leading issues | Modular scale, measure 45–75ch, ≤3 weights, tightened large headings | Computed font-size/line-height/measure (ch); weight count; figure style in tables |
| 3 | **Color & contrast** | Text fails WCAG; rainbow/no system; dark = inverted | Meets AA; palette mostly restrained | Semantic roles, restrained palette, AA+ everywhere, real dark mode | Contrast ratios of text & UI; count of distinct hues; pure-black surfaces in dark |
| 4 | **Spacing & rhythm** | Arbitrary px everywhere; cramped or uneven | Mostly on a scale; a few one-offs | Single scale; consistent section/group gaps; grid-aligned | Gap values between groups vs sections; off-scale px; misalignments |
| 5 | **Consistency** | Buttons/cards/radii/icons all differ | Minor drift | Uniform components, radii, shadows, one icon set | Distinct button styles; radius values; mixed icon stroke/fill |
| 6 | **Polish & detail** | Misaligned, heavy shadows, no hover/focus/empty states | Decent; some states missing | Optical alignment, hairline borders, soft shadows, all states designed | Focus ring presence; shadow opacity/consistency; empty/hover/disabled states |
| 7 | **Brand fit** | Generic "AI slop"; effect-piled; off-personality | On-brand but unremarkable | Distinct, intentional, matches stated direction | Gradient/glass/glow overuse; alignment to direction statement |

## Severity mapping

- **P1 — fix first:** breaks the experience or fails accessibility. Any contrast failure on text, no discernible hierarchy, broken layout at a target viewport, unreadable body size.
- **P2 — clearly cheapens it:** inconsistent components, off-scale spacing, weak heading hierarchy, effect overuse, missing focus/hover states.
- **P3 — refinement:** optical alignment, micro-rhythm, shadow tuning, motion easing, figure styles.

## Finding template (use verbatim shape)

> **P{1|2|3} — {one-line problem}.** ({Dimension} {n}/5[, {Dimension} {n}/5])
> {What's wrong, with measured evidence — sizes, ratios, counts.}
> **Before:** `{current class / value}`
> **After:** `{proposed Tailwind class / token value / numeric step}` → token: {semantic role mapping if relevant}.
> *Impact:* {what this fixes}. Hand to `frontend-design` to apply.

## Process checklist

1. Capture desktop (1440×900) + mobile (390×844); both themes if present; one key interactive state.
2. Pull computed styles for suspect elements so findings cite real values.
3. Score all 7 dimensions with evidence.
4. Convert low scores into findings; rank P1→P3 by impact, not by rubric order.
5. Express every fix in Tailwind/token terms; cross-ref `frontend-design` (implement), `ux` (flow), `performance` (asset weight).
