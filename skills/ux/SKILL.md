---
name: ux
description: Usability, interaction, flow, information architecture, and accessibility for the React Router 7 / Tailwind / TypeScript stack. Use when designing or auditing task flows, forms, navigation/IA, screen state matrices (empty/loading/error/etc.), keyboard & focus behavior, WCAG 2.2 AA accessibility, microcopy, or running a live heuristic UX audit of a running site. NOT visual aesthetics (use `designer`) or token/styling implementation (use `frontend-design`).
---

# UX: usability, interaction, accessibility, audits

You own how the product *works* and *feels to operate* — not how it looks (`designer`) and not how styles are implemented (`frontend-design`). Optimize for: can the user find it, understand it, complete the task without error, and recover when something breaks. Stay concrete: every recommendation must tie to a specific interaction, markup change, copy string, or state.

## Scope routing — read this first
- Look, brand, color palette, type scale, visual hierarchy → hand to `designer`.
- Implementing tokens/components/Tailwind classes → `frontend-design`.
- Building the flow/route/action you spec → `frontend-dev`.
- Perceived speed, skeleton timing, bundle/route-load cost → `performance` (you set the *targets*, they hit them).
- You decide: IA, task flows, state coverage, affordances, focus order, copy, accessibility behavior, audit findings.

## Operating rules
1. **Semantic HTML before ARIA.** A `<button>`, `<a href>`, `<nav>`, `<label>` gives you behavior + a11y for free. Reach for ARIA only to fill a gap native elements can't.
2. **Every screen has six states.** Never spec only the happy path. See the state matrix below.
3. **Color is never the only signal.** Pair every color meaning with text, icon, or shape.
4. **Touch targets ≥ 44×44 px**; primary actions reachable in the thumb zone on mobile.
5. **One primary action per view.** Demote the rest to secondary/tertiary.
6. **Name things by the user's word, not the system's.** Labels and nav reflect user mental models, not table names.
7. **Specify, don't just opine.** Output acceptance criteria and an edge-case list `frontend-dev` can build against.

## Information architecture
- **Navigation model:** pick deliberately — flat (≤7 top-level), hub-and-spoke, hierarchical, or sequential (wizard). Match depth to task frequency: frequent tasks ≤1 click, rare tasks may be nested.
- **Labeling:** concrete nouns/verbs; no clever or internal jargon. Test each label against "would a first-time user predict what's behind it?"
- **Hierarchy:** one primary action, clear content order top→bottom matching the user's reading/decision sequence. Group related controls; separate destructive from constructive.
- **Findability:** provide at least two paths to important content (nav + search, or nav + contextual link). Add breadcrumbs once depth ≥3.
- **Progressive disclosure:** show the 20% everyone needs; tuck the rest behind "Advanced", accordions, or a second step. Disclosure must be discoverable (visible affordance), not hidden.

## Interaction & flow design
- **Map the flow as states + transitions**, not just screens. For each step note: entry condition, user action, system response, success exit, error exit, back/cancel behavior.
- **Forms:**
  - Label every field with a visible `<label htmlFor>`; placeholders are not labels.
  - Group with `<fieldset>/<legend>`; mark required with text ("required"), not color alone.
  - Validate on blur for format, on submit for completeness; show errors inline next to the field, set `aria-invalid` and `aria-describedby` pointing at the error text, and move focus to the first error on failed submit.
  - Right input affordance: native `<select>` / date / `inputmode` / `autocomplete` / `type=email|tel|number` — these unlock mobile keyboards and autofill.
  - Preserve user input across errors and navigation. Never silently clear a form.
- **Feedback & system status (Nielsen #1):** every action gets a visible response within ~100ms. Optimistic for cheap, reversible, near-certain-success mutations; pending UI for slow or failure-prone ones.

### React Router 7 pending/optimistic patterns
- Use `useNavigation()` (`state === "submitting" | "loading"`) to drive global/route pending UI; disable the submit button and show progress while submitting.
- Use `<Form>` actions for mutations; let RR7 revalidate loaders after the action rather than hand-managing refetch.
- Optimistic UI: read `useFetcher().formData` (or `navigation.formData`) to render the expected post-mutation state immediately, then reconcile when the action resolves. Roll back visibly on error.
- Use `useFetcher()` for in-place mutations that should NOT navigate (inline edits, like/favorite, add-to-cart). Show its `fetcher.state` on the specific control, not the whole page.
- Defer slow non-critical data with `defer`/`Await`/`Suspense` so the shell paints fast and only the slow region shows a skeleton.
- On route change, the SPA does NOT reset focus — you must (see Accessibility).

## The six-state matrix — design every one
For each meaningful screen/region, design all that apply:
| State | Design intent |
|---|---|
| **Empty** | First-use vs filtered-empty are different. Explain what goes here + a primary CTA to create/seed it. Never a blank box. |
| **Loading** | Skeleton matching final layout (no jank) for content; spinner only for short indeterminate waits. Set a target: skeleton ≤ ~1s before reassurance. |
| **Partial** | Some data loaded, rest deferred — render what you have, skeleton the rest, never block the whole page. |
| **Error** | Plain-language cause + a recovery action (retry / go back / contact). Preserve user input. Distinguish field error vs page error vs network error. |
| **Success** | Confirm the outcome (toast/inline), show the new state, and offer the obvious next step. |
| **Offline / degraded** | Detect via `navigator.onLine` + failed fetch; tell the user, queue or disable mutations, auto-recover on reconnect. |

