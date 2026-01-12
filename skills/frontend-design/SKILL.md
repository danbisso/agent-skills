---
name: frontend-design
description: Translate a design into production frontend code — define design tokens and wire them into Tailwind v4 @theme, build component variants, do responsive/fluid layout, and match a comp pixel-for-pixel. Use when implementing a Figma/mockup, setting up or auditing a design system, picking semantic vs primitive tokens, choosing @apply vs component vs inline utilities, building variants (cva / tailwind-variants), adding dark mode, or doing a design-to-code review. Stack: React Router 7, Tailwind v4 (CSS-first @theme), TypeScript.
---

# Frontend Design

You are the bridge between a designer's intent and shipped code. Your job is fidelity and durability: the UI matches the comp AND is built so the next change is a token edit, not a rewrite. You are NOT inventing the look (that's `designer`) and NOT wiring data/routes (that's `frontend-dev`).

## Order of operations

When implementing any design, work outside-in in this order. Skipping a step is how you get utility soup and magic numbers.

1. **Extract tokens first.** Before writing one component, pull the design's recurring values into a token scale: color, spacing, type, radius, shadow, z-index, motion. If the comp uses 14 slightly-different greys, that's a token-scale problem to resolve with the designer, not 14 hex codes to transcribe.
2. **Wire tokens into `@theme`** so every Tailwind utility derives from a token. No raw hex, no `p-[13px]`, no `text-[#1a1a1a]` in component code.
3. **Build primitives, then compose.** Button/Input/Card/Stack before pages.
4. **Add variants** for the axes the design actually varies on (intent, size, state) — not speculative ones.
5. **Make it responsive/fluid**, mobile-first.
6. **Run the design-to-code review** (see `design-review-checklist.md`).

## Design tokens are the single source of truth

Two layers. Keep them separate.

- **Primitive tokens** = raw scale values, named by what they ARE: `--color-blue-500`, `--space-4`, `--text-lg`. No meaning, just a palette.
- **Semantic tokens** = roles, named by what they DO: `--color-bg`, `--color-fg`, `--color-accent`, `--color-border`, `--color-danger`. They *reference* primitives. Components consume semantic tokens only.

This indirection is what makes dark mode and re-skinning a one-file change. A `Button` should never know it is blue; it knows it is `accent`.

### Token scale discipline

- **Color:** primitives in perceptual steps (50–950). Semantic roles: `bg`, `bg-muted`, `bg-subtle`, `fg`, `fg-muted`, `border`, `accent`/`accent-fg`, plus status `danger`/`warning`/`success`/`info` each with a `-fg` pair for accessible text on top.
- **Spacing:** one geometric/linear scale (4px base is conventional). Everything — padding, gap, margin — pulls from it. No off-scale values.
- **Type:** a modular scale (e.g. ratio ~1.2) with paired `line-height` per step. Ship font-size and line-height together.
- **Radius / shadow / z-index / motion:** small named scales (`sm/md/lg`, plus `--ease-*` and `--duration-*`). Z-index especially: name your layers (`--z-dropdown`, `--z-modal`, `--z-toast`) so stacking is intentional, never a `9999` arms race.

## Tailwind v4: wire tokens into `@theme`

In v4 the config is CSS-first. Anything you declare in `@theme` becomes both a CSS variable AND a generated utility. This is the mechanism that makes utilities derive from tokens.

```css
/* app.css */
@import "tailwindcss";

@theme {
  /* primitives — generate color-blue-500, etc. */
  --color-blue-500: oklch(0.62 0.19 256);
  --color-neutral-50: oklch(0.985 0 0);
  --color-neutral-900: oklch(0.205 0 0);

  /* type: paired size + line-height */
  --text-sm: 0.875rem;
  --text-sm--line-height: 1.25rem;
  --text-base: 1rem;
  --text-base--line-height: 1.5rem;

  /* radius / motion / layers */
  --radius-md: 0.5rem;
  --ease-out-smooth: cubic-bezier(0.22, 1, 0.36, 1);
  --duration-fast: 150ms;
}

/* semantic layer: roles reference primitives, switch per theme.
   Use a non-@theme block so these aren't emitted as primitive utilities. */
:root {
  --color-bg: var(--color-neutral-50);
  --color-fg: var(--color-neutral-900);
  --color-accent: var(--color-blue-500);
}
.dark {
  --color-bg: var(--color-neutral-900);
  --color-fg: var(--color-neutral-50);
}

/* expose semantic roles as utilities (bg-bg, text-fg, ...) */
@theme inline {
  --color-bg: var(--color-bg);
  --color-fg: var(--color-fg);
  --color-accent: var(--color-accent);
}
```

`@theme inline` emits utilities that reference the variable rather than inlining its value, so `.dark { --color-bg: ... }` actually re-themes `bg-bg`. Use plain `@theme` for static primitives, `@theme inline` for anything that changes per theme/context.

**Dark mode:** drive it off a `.dark` class (toggle on `<html>`) and a `@custom-variant dark (&:where(.dark, .dark *));` so `dark:` variants work. Prefer flipping semantic variables over scattering `dark:` utilities everywhere — define the role once, theme it in one place.

### @apply vs component class vs inline utility

Decide by reuse and ownership, in this priority:

1. **Inline utilities (default).** One-off layout and the common case. Co-located, greppable, no naming tax.
2. **A component (React) with variants.** The right home for a *reused* visual element. The abstraction is the component, not a CSS class. This is where 90% of "I keep repeating these classes" goes.
3. **`@apply` in a `@layer components` block — sparingly.** Only for things you can't componentize: styling third-party/markdown HTML you don't control (`.prose a`), or base element resets. Never `@apply` to avoid making a component — that just hides utility soup behind a class name and loses variant ergonomics.

Smell of utility soup: the same 8-class string copy-pasted across files. Fix = a variant'd component, not a `.btn` class.

## Component architecture for fidelity

- **Split presentational from controlled.** A presentational `<Button>` takes props and renders; state/data live in the caller (`frontend-dev`'s territory). Design-fidelity components stay dumb and reusable.
- **Composable primitives + slots.** Build `Card` as `Card`/`Card.Header`/`Card.Body` (or `asChild`/slot patterns) so consumers compose rather than receive 20 props.
- **Variants encode the design's axes.** Use `tailwind-variants` or `cva`. Define the matrix once.

```ts
import { cva, type VariantProps } from "class-variance-authority";

const button = cva(
  "inline-flex items-center justify-center gap-2 rounded-md font-medium " +
  "transition-colors duration-fast ease-out-smooth focus-visible:outline-none " +
  "focus-visible:ring-2 focus-visible:ring-accent disabled:opacity-50 disabled:pointer-events-none",
  {
    variants: {
      intent: {
        primary: "bg-accent text-accent-fg hover:bg-accent/90",
        ghost: "bg-transparent text-fg hover:bg-bg-muted",
        danger: "bg-danger text-danger-fg hover:bg-danger/90",
      },
      size: { sm: "h-8 px-3 text-sm", md: "h-10 px-4 text-base", lg: "h-12 px-6 text-base" },
    },
    defaultVariants: { intent: "primary", size: "md" },
  },
);

export type ButtonProps = VariantProps<typeof button> &
  React.ButtonHTMLAttributes<HTMLButtonElement>;

export function Button({ intent, size, className, ...props }: ButtonProps) {
  return <button className={button({ intent, size, className })} {...props} />;
}
```

Note every class references a semantic token (`bg-accent`, `text-fg`, `ring-accent`, `duration-fast`) — never a literal. That's the payoff of doing tokens first.

## Responsive & fluid

- **Mobile-first.** Author the small layout unprefixed; add `sm:`/`md:`/`lg:` only to escalate. Never the reverse.
- **Breakpoints are layout-change points**, not device widths. Add one when the layout *needs* to reflow, not because a phone is 390px.
- **Fluid type/space with `clamp()`** so you interpolate between breakpoints instead of stepping. Register fluid steps as tokens:

```css
@theme {
  /* clamp(min, preferred(vw-based), max) */
  --text-fluid-lg: clamp(1.25rem, 1.1rem + 0.75vw, 1.75rem);
  --space-fluid-section: clamp(3rem, 2rem + 6vw, 8rem);
}
```

- **Container queries over viewport queries** for components that live in variable-width slots (sidebar vs main). Tailwind v4 has `@container` + `@sm:`/`@md:` container variants. A `Card` should respond to *its container*, not the page — that's what makes it drop-in reusable.

## Pixel & visual fidelity

When matching a comp:

- **Lock the spacing rhythm to the scale.** If the comp shows 13px, it almost always means 12 or 16 — round to a token and confirm. Off-scale values are usually export noise.
- **Optical alignment over mathematical.** Icons next to text, glyphs in circular buttons, and capital letters often need a 1px nudge to *look* centered. Trust the eye.
- **Hit targets ≥ 44×44px** for anything tappable, even when the visual glyph is smaller — pad the interactive element, not the icon.
- **Build every state, not just the resting one.** For each interactive element ship: `hover`, `focus-visible`, `active`, `disabled`, `loading`, and where relevant `empty` and `error`. A design that looks done but has no focus ring or loading state is not done.
- **Type:** match font-size *and* line-height *and* letter-spacing; the last is the one people forget and it's why text "looks off."

