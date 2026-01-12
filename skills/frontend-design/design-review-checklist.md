# Design-to-Code Review Checklist

Audit an implementation against design intent. Go top to bottom; a single ✗ in Tokens usually explains many ✗ downstream. Open the comp and the running UI side by side.

## 1. Tokens (the root)
- [ ] No raw hex / rgb in component code — colors come from semantic tokens (`bg-accent`, not `bg-[#…]`).
- [ ] No arbitrary spacing (`p-[13px]`, `mt-[7px]`) — every value is on the spacing scale.
- [ ] No off-scale font sizes; type uses paired size + line-height tokens.
- [ ] Components consume **semantic** tokens (`bg`, `fg`, `accent`), never primitives (`blue-500`) directly.
- [ ] Token names mirror the designer's named styles (no drift).

## 2. Fidelity to comp
- [ ] Spacing rhythm matches; suspicious values rounded to the scale and confirmed.
- [ ] Type matches on size, line-height, **and** letter-spacing.
- [ ] Optical alignment checked (icons, glyphs, caps) — looks centered, not just measures centered.
- [ ] Border radius, shadows, and borders match the comp's named scale.
- [ ] Color contrast meets WCAG AA (4.5:1 body, 3:1 large/UI) — verify, don't assume.

## 3. States (each interactive element)
- [ ] `hover`
- [ ] `focus-visible` — a visible ring/outline; never `outline: none` without a replacement.
- [ ] `active`
- [ ] `disabled` — visually and via `disabled`/`aria-disabled`, pointer-events off.
- [ ] `loading` — spinner/skeleton, control disabled, layout doesn't jump.
- [ ] `empty` and `error` where the element renders data.
- [ ] Hit target ≥ 44×44px for tappable elements (pad the control, not the glyph).

## 4. Responsive / fluid
- [ ] Authored mobile-first (base unprefixed, escalate with `sm:`/`md:`).
- [ ] No horizontal overflow at 320px; no awkward dead zones between breakpoints.
- [ ] Fluid type/space via `clamp()` where the design interpolates rather than steps.
- [ ] Components in variable-width slots use container queries (`@container`), not viewport breakpoints.
- [ ] Touch and pointer both work (no hover-only affordances).

## 5. Architecture / durability
- [ ] Reused visuals are variant'd components, not copy-pasted class strings (no utility soup).
- [ ] `@apply` used only for uncontrolled HTML (prose/markdown) or base resets — not to dodge a component.
- [ ] Variants encode real design axes (intent/size/state), no speculative ones.
- [ ] Presentational components are stateless; data/state live in the caller.
- [ ] Dark mode (if in scope) is driven by flipping semantic vars, not scattered `dark:` utilities.
- [ ] Z-index uses named layer tokens, not magic `9999`.

## 6. Handoffs triggered?
- [ ] Visual decision missing/ambiguous → `designer`.
- [ ] Flow / a11y semantics / IA question → `ux`.
- [ ] Needs route/loader/action/data wiring → `frontend-dev`.
- [ ] Component sprawl or prop explosion → `clean-code`.
