# Web Vitals cheatsheet

Thresholds + the usual cause→fix for each metric. Companion to `SKILL.md`. Targets are the "good" bucket (the 75th percentile of field visits must beat these).

## Thresholds

| Metric | Good | Needs work | Poor | Where it lives |
|---|---|---|---|---|
| **LCP** (Largest Contentful Paint) | ≤ 2.5s | 2.5–4.0s | > 4.0s | lab + field |
| **INP** (Interaction to Next Paint) | ≤ 200ms | 200–500ms | > 500ms | **field only** (lab proxy: TBT) |
| **CLS** (Cumulative Layout Shift) | ≤ 0.1 | 0.1–0.25 | > 0.25 | lab + field |
| **TTFB** (Time to First Byte) | ≤ 0.8s (aim ≤ 0.2s) | 0.8–1.8s | > 1.8s | lab + field |
| **FCP** (First Contentful Paint) | ≤ 1.8s | 1.8–3.0s | > 3.0s | lab + field |
| **TBT** (Total Blocking Time) | ≤ 200ms | 200–600ms | > 600ms | lab only |

**INP replaced FID** in March 2024. FID measured only input *delay*; INP measures the full interaction→next-paint latency across the whole visit and is far harder to fake. Do not report FID.

LCP is composed of: TTFB + resource load delay + resource load time + render delay. Diagnose by splitting it into these four sub-parts (the trace shows each) — the largest sub-part is where to spend effort.

## Cause → fix per metric

### LCP too high
| Likely cause | Fix |
|---|---|
| Slow TTFB (server/loader) | Edge-cache HTML, parallelize loader queries (`Promise.all`), stream/defer slow data, index queries |
| LCP image discovered late | `fetchpriority="high"`, `<link rel="preload" as="image">`, no `loading="lazy"` on it |
| LCP image too big / wrong format | Responsive `srcset`/`sizes`, AVIF/WebP, correct dimensions |
| Render-blocking CSS/JS | Inline critical CSS, `defer`/`async` scripts, code-split |
| Web font blocking text paint | `font-display: swap`, preload the critical font, subset |

### INP too high (and TBT in lab)
| Likely cause | Fix |
|---|---|
| Long tasks on the main thread | Break up >50ms tasks; `scheduler.yield()`/`setTimeout` to yield; move heavy compute to a Web Worker |
| Large/expensive re-renders on interaction | Profile wasted renders, memoize where it pays, virtualize long lists, stable keys |
| Heavy third-party JS | Defer/lazy-load, load post-interaction, remove if unused |
| Hydration cost | Smaller payloads, fewer client components, stream/defer non-critical subtrees |

### CLS too high
| Likely cause | Fix |
|---|---|
| Images/video without dimensions | Set `width`/`height` or `aspect-ratio` to reserve the box |
| Web font swap reflow (FOUT) | Match fallback metrics with `size-adjust`/`ascent-override`; preload font |
| Ads/embeds/iframes injected late | Reserve space with a min-height placeholder |
| Content inserted above existing content | Reserve space; never push content down after paint |
| Animating layout properties | Animate `transform`/`opacity`, never `top`/`width`/`height` |

### TTFB too high
| Likely cause | Fix |
|---|---|
| Serial loader query waterfall | `Promise.all` independent queries; push data to the deepest route that needs it |
| Slow/unindexed DB queries | Index filter/sort/join columns; `EXPLAIN QUERY PLAN`; select only needed columns |
| No edge caching | Cache HTML/loader responses at the edge where freshness allows |
| Cold starts (non-Workers infra) | Hand off to `aws-architect` — provisioned concurrency / warm pools |

### FCP too high
| Likely cause | Fix |
|---|---|
| Render-blocking stylesheet | Inline critical CSS; verify Tailwind purge `content` globs |
| Render-blocking head scripts | `defer`/`async`; move below the fold |
| Slow TTFB upstream | Fix TTFB first (FCP can't beat TTFB) |

## Measurement conditions (keep constant across runs)
- **Production build**, served like production (never the dev server).
- **Throttling:** 4× CPU slowdown + Slow-4G (mobile baseline) unless explicitly testing desktop.
- **≥3 runs, report the median.** Single runs are noise.
- **Same URL, same viewport, same throttle** before vs after — otherwise the delta is meaningless.
- **Validate in the field** (CrUX / `web-vitals` RUM) after shipping; lab is the microscope, field is the truth.
