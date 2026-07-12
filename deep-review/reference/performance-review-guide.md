# Performance Review Guide

Reminder checklist of performance signals to catch in a diff — frontend, JS, memory, database, API, and algorithmic complexity. Assumes you know the fundamentals; this lists what to flag.

## Frontend (Core Web Vitals)

Targets: LCP ≤ 2.5s · INP ≤ 200ms (replaced FID in 2024) · CLS ≤ 0.1 · FCP ≤ 1.8s · TBT ≤ 200ms. "Poor" starts at: LCP > 4s, INP > 500ms, CLS > 0.25, FCP > 3s. JS bundle: good < 200KB, poor > 500KB.

- LCP image with `loading="lazy"` → flag; it should have `fetchpriority="high"` instead.
- Large PNG/JPG heroes → suggest AVIF/WebP via `<picture>`, responsive srcset; check SSR/SSG and CDN for critical content.
- Render-blocking full stylesheet or fonts → inline critical CSS, `rel="preload"` the rest, `font-display: swap` on `@font-face`.
- Long synchronous work in event handlers → chunk it and yield to the main thread, or move to a Web Worker. The yield idiom is easy to mis-write:

  ```javascript
  // Parens required: `await a ?? b` parses as `(await a) ?? b`,
  // so the setTimeout fallback would never be awaited.
  await (globalThis.scheduler?.yield?.() ?? new Promise(r => setTimeout(r, 0)));
  ```

- CLS: images/videos without width/height or `aspect-ratio`; dynamic content (ads, banners) without reserved `min-height`; content inserted above existing content.
- CSS animations: only `transform`/`opacity` are compositor-friendly. Flag animating width/height/top/left/margin, and any `transition: all` (list properties explicitly).

## JavaScript Performance

- Heavy components/routes imported statically → `lazy(() => import(...))`, route-level code splitting.
- Whole-library imports (`import _ from 'lodash'`, `moment`) → per-function imports / lighter alternatives (date-fns).
- Default-exported object of functions defeats tree shaking → named exports.
- Bundle not analyzed / unused dependencies → suggest webpack-bundle-analyzer.
- Lists > 100 items rendered directly → virtualize (react-window); tables need pagination or virtualization; watch for unnecessary full re-renders.

## Memory Management

Full React guidance: [react.md](react.md).

- `useEffect` with no cleanup return → flag missing `removeEventListener`, `clearInterval`/`clearTimeout`, `ws.close()` (WebSocket/SSE), subscription unsubscribes.
- Closures capturing large objects when only a derived value is needed → extract the value before returning the closure.
- Global variables/collections accumulating data without release.
- Tools to suggest: Chrome DevTools Memory (heap snapshots), MemLab (automated leak detection), Performance Monitor.

## Database Performance

- N+1: query inside a loop over a parent result set → eager load (`select_related`/`prefetch_related`, TypeORM `relations`) or batch with `WHERE id IN (...)`.
- Index-defeating predicates: function applied to a column (`YEAR(created_at) = 2024` → range compare) and leading-wildcard `LIKE '%x%'` (prefix match `'x%'` is indexable).
- WHERE-clause columns without indexes; composite index column order wrong; unused indexes lingering.
- `SELECT *` where specific columns suffice; queries on large tables without `LIMIT`/pagination.
- Recommend `EXPLAIN` on new/changed queries and slow-query log monitoring.

## API Performance

- List endpoints returning everything → require pagination with an enforced max page size (e.g. `Math.min(limit, 100)`).
- Hot data uncached → Redis with an expiration (`setex`), or HTTP caching (`Cache-Control`, `ETag`).
- No response compression (gzip/Brotli); responses returning fields the client doesn't need (support field selection).
- Rate limiting is owned by [security-review-guide.md → Rate Limiting](security-review-guide.md); from the performance angle just confirm limits exist so one client can't exhaust capacity.

## Algorithmic Complexity

Scale intuition: O(n²) at 1000 items = 1M ops; at 1M items = 1T ops.

- Nested loops over the same collection, `Array.includes`/`indexOf`/`find` inside a loop → O(n²); use a `Set`/`Map` for O(1) membership and keyed lookup.
- Repeated `array.find(u => u.id === id)` lookups → build a `Map` once.
- Deep recursion (O(n) stack) on unbounded input → risk of stack overflow; suggest iteration.
- Avoidable O(n) copies (`map` where in-place mutation is acceptable) on hot paths.
- Comment phrasing: name the complexity and the fix ("`includes` in a loop makes this O(n²); use a Set").

## Low-Level Efficiency Anti-Patterns

Resource-lifecycle defects (unclosed connections, listeners, timers) live in [common-bugs-checklist.md → Resource Management](common-bugs-checklist.md#resource-management).

- Loop-invariant work inside the loop (re-reading files/config, re-parsing, recompiling regexes) → hoist it out; cache or pass computed results downstream.
- Independent async operations `await`ed sequentially → `Promise.all` / `asyncio.gather` / `tokio::join!`.
- Hot-path bloat: heavy work at module/import time, per-request initialization that could be deferred, startup work blocking the first request.
- Unbounded growth: global dicts/caches without max-size or TTL; queues/logs/metrics buffers without an upper bound; long-lived references pinning per-request objects. Fix pattern:

  ```python
  @lru_cache(maxsize=256)   # bounded, not a bare module-level dict
  def get_cached(key): ...
  ```

## Metric Thresholds

| Metric | Good | Poor |
|---|---|---|
| LCP / INP / CLS / FCP | ≤ 2.5s / ≤ 200ms / ≤ 0.1 / ≤ 1.8s | > 4s / > 500ms / > 0.25 / > 3s |
| JS bundle | < 200KB | > 500KB |
| API response | < 100ms | > 500ms |
| DB query | < 50ms | > 200ms |
| Page load | < 3s | > 5s |

## Tools

Lighthouse / WebPageTest (Web Vitals), webpack-bundle-analyzer, Chrome DevTools Performance & Memory, MemLab, EXPLAIN, pganalyze, New Relic / Datadog.

## Review Checklist

**Blocking:**
- [ ] LCP image not lazy-loaded; no `transition: all`; no animation of width/height/top/left
- [ ] Lists > 100 items virtualized
- [ ] No N+1 queries; list endpoints paginated; no `SELECT *` on large tables
- [ ] No O(n²)+ nested loops on unbounded data
- [ ] useEffect/listeners/timers/sockets have cleanup

**Recommended:**
- [ ] Code splitting; on-demand library imports; WebP/AVIF images; no unused dependencies
- [ ] Hot data cached; WHERE columns indexed; slow-query monitoring
- [ ] Response compression; rate limiting present; only needed fields returned
- [ ] Independent awaits parallelized; loop-invariant work hoisted; caches bounded (max-size/TTL)

**Nice to have:**
- [ ] Bundle size analyzed; CDN in use; performance monitoring; benchmarks run
