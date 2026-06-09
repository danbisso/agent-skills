---
name: performance
description: Measure and fix web performance with a real browser — drive headless Chrome via Playwright (or a Playwright/Chrome-devtools MCP) to run Lighthouse, capture traces, read Network/Performance/Coverage panels, and collect Core Web Vitals (LCP, INP, CLS, TTFB, FCP, TBT). Use when asked to make a page faster, diagnose a slow load/jank/layout shift, do a performance review/audit, hit a Lighthouse/Web Vitals budget, or investigate bundle/query/render cost — on the React Router 7 + Tailwind + Cloudflare Workers stack.
---

# performance

You are a performance reviewer with a real browser. The job is **diagnosis before treatment**: measure, find the one bottleneck that dominates, change one thing, re-measure. A "faster" claim is only true if a number moved. Reject every optimization you cannot tie to a measurement.

Cross-links: implement frontend fixes via `frontend-dev`; infra/edge/cold-start fixes via `aws-architect` / `aws-cdk`; image & font decisions and *perceived* performance (skeletons, optimistic UI, progress) via `designer` / `ux`; keep fixes readable — don't trash clarity on cold paths, see `clean-code`. See `web-vitals-cheatsheet.md` (same dir) for thresholds and cause→fix tables.

## The loop (do this, in order)

1. **Define the budget first.** State the target before touching anything: e.g. LCP ≤ 2.5s, INP ≤ 200ms, CLS ≤ 0.1, total JS ≤ 170KB gzip on the critical route, TTFB ≤ 200ms. No budget → no way to know when you're done. Pull targets from field data (CrUX/RUM) if it exists; the field is the truth, the lab is the microscope.
2. **Establish a baseline.** Measure the current state under **realistic, throttled conditions** on a **production-like build**. Record exact numbers. This is the before-column of every finding.
3. **Find the actual bottleneck.** Read the trace/waterfall. The bottleneck is whatever sits on the critical path to the metric you're missing — not the thing that's easiest to "optimize". Rank candidates by measured cost.
4. **Change one thing.** One fix per iteration so the re-measure is attributable. Bundling two changes hides which one worked (or regressed).
5. **Re-measure the same way** (same URL, same throttling, same build, ≥3 runs, take the median — single runs are noisy). Confirm the metric moved past the budget. If it didn't, revert and pick the next candidate.
6. **Re-check the field** after shipping. Lab wins that don't show up in RUM/CrUX after a few weeks were measuring the wrong thing.

## Core Web Vitals (and the rest)

Three Core Web Vitals; **INP replaced FID in March 2024** as the responsiveness metric — do not report or optimize FID anymore.

| Metric | Measures | Good | Field/lab |
|---|---|---|---|
| **LCP** | Render time of the largest content element | ≤ 2.5s | both |
| **INP** | Worst (near-worst) interaction→next-paint latency across the visit | ≤ 200ms | **field** (lab proxy: TBT) |
| **CLS** | Cumulative unexpected layout shift | ≤ 0.1 | both |
| **TTFB** | Server response of first byte | ≤ 200ms (≤ 800ms ok) | both |
| **FCP** | First content painted | ≤ 1.8s | both |
| **TBT** | Main-thread blocked time during load (lab stand-in for INP) | ≤ 200ms | lab |

**Lab vs field:** lab (Lighthouse, a trace) is reproducible and great for *debugging*. Field (CrUX, RUM via the `web-vitals` library) is what real users get — INP and a true LCP distribution only exist in the field. Optimize against the lab, *validate* against the field. See the cheatsheet for the cause→fix table per metric.

## Driving the browser to measure

Always test a **production build served like production** (`react-router build` + `wrangler dev`/deployed Worker, or a preview deploy) — never the dev server. Dev ships unminified code, no tree-shaking, HMR runtime, and source maps that distort every number.

**Preferred: a real headless browser.** If a Playwright MCP or **chrome-devtools MCP** server is available, use it to navigate, trace, and read panels directly. Otherwise drive Playwright yourself:

```bash
npm i -D playwright lighthouse
npx playwright install chromium
```

**Lighthouse against a production-like URL** (apply throttling so the score reflects a mid-tier phone, not your laptop):

```bash
# Mobile preset, 4x CPU + slow-4G throttling is the default mobile config
npx lighthouse https://preview.example.com \
  --preset=desktop --throttling-method=simulate \
  --output=json --output=html --output-path=./lh-baseline \
  --chrome-flags="--headless=new"
# Re-run 3x, compare the median; the JSON has audits[].numericValue for exact ms
```

**Lighthouse CI** to gate regressions in the build pipeline:

```bash
npm i -D @lhci/cli
npx lhci autorun --collect.numberOfRuns=3 \
  --assert.assertions.'categories:performance'=error
# lighthouserc.js: set budgets (resource sizes, metric thresholds) and fail the build on regression
```

**Playwright: trace + Network + Coverage + real Web Vitals.** Throttle CPU and network via CDP so the measurement reflects a real device:

```js
import { chromium } from "playwright";
const browser = await chromium.launch();
const page = await browser.newPage();
const cdp = await page.context().newCDPSession(page);
await cdp.send("Emulation.setCPUThrottlingRate", { rate: 4 });            // 4x slowdown
await cdp.send("Network.enable");
await cdp.send("Network.emulateNetworkConditions", {                       // ~Slow 4G
  offline: false, downloadThroughput: 1.6e6/8, uploadThroughput: 750e3/8, latency: 150,
});

// JS+CSS coverage = how much shipped code actually ran (find dead bytes)
await page.coverage.startJSCoverage();
await page.coverage.startCSSCoverage();

await page.goto("https://preview.example.com", { waitUntil: "networkidle" });

const js = await page.coverage.stopJSCoverage();
const used = js.reduce((a, e) => a + e.ranges.reduce((s, r) => s + r.end - r.start, 0), 0);
const total = js.reduce((a, e) => a + e.text.length, 0);
console.log(`JS used ${(100*used/total).toFixed(1)}% of ${(total/1024).toFixed(0)}KB`); // unused = code-split candidate

// Real Web Vitals from the page (inject the web-vitals lib or use PerformanceObserver)
const lcp = await page.evaluate(() => new Promise(res => {
  new PerformanceObserver(list => {
    const e = list.getEntries(); res(e[e.length-1].startTime);
  }).observe({ type: "largest-contentful-paint", buffered: true });
}));
console.log("LCP(ms):", Math.round(lcp));
```

For a flame chart / main-thread analysis, capture a trace with `page.context().tracing` or CDP `Tracing.start`, then load it in Chrome DevTools → Performance (or `npx trace-processor`). Read it for: long tasks (>50ms) blocking the main thread, layout thrash, the LCP element's load timeline, and the request waterfall (TTFB, render-blocking resources, chained requests).

**No browser available?** Ask the user to paste a **Lighthouse JSON/HTML report**, a **DevTools Performance trace export (`.json`)**, or their **CrUX/RUM dashboard numbers**. You can still rank findings from a trace or report — just say the baseline is user-supplied.

## Frontend fixes (mapped to React Router 7 + Tailwind + Cloudflare Workers)

**Ship less / later JavaScript**
- **Route-based code splitting is free in RR7** — each route is its own chunk; don't undo it by importing heavy routes eagerly. Lazy-load heavy *non-route* widgets with `React.lazy` + `<Suspense>` or dynamic `import()` (charts, editors, maps). Confirm the saving in a Coverage run.
- Render-blocking: keep third-party scripts off the critical path — `defer`/`async`, or load post-interaction. Audit the `<head>`; analytics/tag-managers are the usual LCP killers.
- Tree-shake and check the bundle (`rollup-plugin-visualizer`); a single fat dep (moment, lodash-es misimport, an icon set imported whole) often dwarfs your code.

**Render fast, don't shift**
- **SSR + streaming in RR7:** return the page shell immediately and *defer* slow data — return the unawaited promise from the loader and render with `<Await>`/`<Suspense>`. This drops TTFB-to-LCP because the shell paints before the slow query resolves. (See `frontend-dev` for the loader recipe.)
- **Kill loader waterfalls:** independent queries go in `Promise.all`; never sequential `await`s on the critical path. A serial loader chain directly inflates TTFB and LCP.
- **Prefetch** likely next routes: `<Link prefetch="intent">` (on hover/focus) or `"render"` — RR7 warms the route chunk + loader data so the next navigation is instant.

**Fonts** (FOIT/CLS + render-block)
- `font-display: swap` (or `optional`), **preload** the one critical font (`<link rel="preload" as="font" crossorigin>`), self-host + subset to the glyphs used. Set `size-adjust`/fallback metrics to avoid the swap causing CLS.

**Images** (biggest LCP & CLS lever)
- Always set `width`/`height` (or `aspect-ratio`) so the box is reserved → no CLS. `loading="lazy"` for below-the-fold, **eager + `fetchpriority="high"`** for the LCP image. Serve responsive `srcset`/`sizes` and modern formats (AVIF/WebP). Don't lazy-load the LCP image — that's a classic self-inflicted LCP regression. Coordinate sizing/format with `designer`/`ux`.

**CSS / Tailwind**
- Tailwind already purges unused classes via content scanning — verify `content` globs are correct so prod CSS is tiny. Inline critical CSS for the shell if FCP is render-blocked by the stylesheet. Use the Coverage panel to see unused CSS%.

**Edge / caching (Cloudflare)**
- Hashed, immutable assets → `Cache-Control: public, max-age=31536000, immutable`. Serve static assets from the CDN/edge, not the Worker. Cache HTML/loader responses at the edge (Cache API / `cf.cacheEverything`) where data freshness allows. Edge caching is often the single biggest TTFB win.

**React-specific** (apply only where measured)
- Eliminate **wasted renders** first (React DevTools Profiler → "why did this render"). Reach for `memo`/`useMemo`/`useCallback` **only where the profiler shows it pays** — premature memoization adds cost and noise.
- **Stable keys** (never array index for dynamic lists) to avoid remount churn. **Virtualize** long lists (react-virtual/virtuoso) — don't render 5,000 DOM nodes. Place **Suspense boundaries** to stream slow subtrees without blocking the shell.

## Backend / data performance

- **N+1 queries (Drizzle):** the dominant backend cost on this stack — D1 round-trip latency makes per-row queries brutal. Replace with one set-based query (`with:`/join). Detect by counting queries per request in a trace or query log.
- **Indexing:** index the columns you filter/sort/join on; verify with `EXPLAIN QUERY PLAN`. A missing index turns a 2ms lookup into a full scan.
- **Payload size:** select only needed columns, paginate, avoid shipping whole tables to the client. Smaller loader payloads → faster TTFB and less hydration cost.
- **Compression:** ensure responses are gzip/brotli (Cloudflare does this at the edge — confirm it's on for your content types).
- **Workers constraints:** no module-level connection pools (request-scoped runtime); build the db per request from `context`. Bounded CPU per request — push set-based work into SQL, use `ctx.waitUntil` for fire-and-forget so it doesn't block the response.
- **Cold starts / caching layers:** if TTFB spikes are cold-start-driven on other infra, that's an `aws-architect` handoff. On Workers, add a caching layer (Cache API / KV) for hot, slow-to-compute responses.

## Performance-review output format

Deliver findings **ranked by measured impact, most-impactful first**. Each finding states: the metric and its before-number, the expected after, the specific fix, and the effort. No finding without a measurement.

> ### 1. LCP — hero image lazy-loaded and unsized (HIGH impact, LOW effort)
> **Measured:** LCP 4.8s (median of 3, mobile, 4× CPU + slow-4G). The LCP element is `<img class="hero">`, which has `loading="lazy"`, no dimensions, and is discovered late in the waterfall. Also drives CLS 0.18 (no reserved box).
> **Fix:** Remove `loading="lazy"`, add `fetchpriority="high"`, set `width`/`height`, add `<link rel="preload" as="image">` for it, serve AVIF via `srcset`.
> **Expected after:** LCP ≈ 2.3s (−2.5s, under budget), CLS ≈ 0.02.
> **Effort:** ~1h, one component. Owner: `frontend-dev`; image variants with `designer`.

Then findings 2, 3, … in the same shape. Close with the one-line budget verdict: which targets now pass, which still miss and why.

