# Per-screen accessibility pre-ship checklist (WCAG 2.2 AA)

Run this against each screen before shipping. Fail = block or log a severity-rated finding (see SKILL.md rubric).

## Structure & semantics
- [ ] Exactly one `<h1>`; heading levels descend without skips.
- [ ] Landmarks present: `<header> <nav> <main> <footer>`; one `<main>`.
- [ ] Lists use `<ul>/<ol>`, tabular data uses `<table>` with `<th scope>`.
- [ ] Buttons are `<button>`, links are `<a href>` — not clickable `<div>`s.
- [ ] Skip-to-content link is the first focusable element.

## Keyboard & focus
- [ ] Every interactive element reachable by Tab; order matches visual order.
- [ ] Visible focus indicator on all focusable elements (≥3:1 contrast).
- [ ] No keyboard traps; Esc closes dialogs/menus.
- [ ] SPA route change moves focus to new view (`<h1>` or region) and announces it.
- [ ] Dialog/menu close returns focus to the trigger.
- [ ] Focus not hidden behind sticky headers (WCAG 2.4.11).

## Names, roles, values
- [ ] Every control has an accessible name (icon-only buttons included).
- [ ] Decorative images `alt=""`; meaningful images have descriptive alt.
- [ ] ARIA used only where native semantics fall short; no redundant/incorrect roles.
- [ ] Custom widgets expose correct role + state (`aria-expanded`, `aria-selected`, etc.).

## Forms
- [ ] Visible `<label htmlFor>` for every field (placeholder is not a label).
- [ ] Required marked with text, not color alone; `aria-required` set.
- [ ] Errors: inline text + `aria-invalid` + `aria-describedby` to the message.
- [ ] Focus moves to first invalid field on failed submit.
- [ ] Correct `type` / `inputmode` / `autocomplete` for mobile keyboards & autofill.
- [ ] Input preserved across validation errors and navigation.

## Perception
- [ ] Text contrast ≥4.5:1 (≥3:1 large/bold); UI & focus ≥3:1. (Flag fails to `designer`.)
- [ ] Meaning never by color alone — pair with text/icon/shape.
- [ ] `prefers-reduced-motion` honored; non-essential motion disabled.
- [ ] Content readable & operable at 200% zoom and 320px width (reflow, no horizontal scroll).

## Dynamic & status
- [ ] Async status (toasts, validation, result counts) in `aria-live`.
- [ ] Loading states announced, not silent.
- [ ] Target size ≥24px min, prefer 44×44px (WCAG 2.5.8).
- [ ] Drag actions have a single-pointer alternative (WCAG 2.5.7).

## Verify with tools
- [ ] Automated pass: axe-core / Lighthouse a11y (catches ~30–40% of issues).
- [ ] Manual keyboard-only run of each critical flow.
- [ ] Screen-reader spot check of the primary task (VoiceOver/NVDA).
