# rd-performance-articles

**R&D: Performance and architecture patterns** — a series of articles on frontend performance, covering Core Web Vitals, rendering strategies (SSR/SSG/ISR), hydration, multi-layer caching, and bundle analysis.

The series is written as a coherent path: it starts by defining the metrics that matter, then works through the architectural decisions that move those metrics — how and where HTML is produced, how responses are cached, and how much JavaScript ends up on the critical path.

## Articles

### 1. Core Web Vitals: A Coordinate System for Performance Optimization
Establishes the shared vocabulary and target thresholds used throughout the series. Explains the three metrics — LCP (loading), INP (responsiveness), and CLS (visual stability) — the difference between lab and field data, common causes of degradation, and concrete optimization practices for each metric.

### 2. Rendering Strategies and Hydration
Treats rendering models and hydration as one topic. Compares CSR, SSR, SSG, and ISR (where and when HTML is produced, and the trade-offs of each), then explains hydration as a distinct phase that affects INP and TTI, including hydration mismatches and modern approaches such as streaming SSR with Suspense.

### 3. Caching at All Levels
Presents caching as a multi-layer architecture rather than isolated tricks. Walks through the browser cache and client-side data layer, the CDN and transport level, service workers, and the origin/data layer — unified by the stale-while-revalidate principle — and tackles the hardest problem: cache invalidation.

### 4. Bundle Analysis and Code Splitting
Covers measuring and reducing JavaScript that reaches the client. Explains bundle analysis and treemap reading, route- and component-level code splitting, tree shaking, and the architectural trade-offs each decision introduces to keep code off the critical path.

## Recurring Themes

Across the series, each topic is tied back to measurable outcomes (LCP, INP, CLS, TTFB) and framed around explicit architectural trade-offs, so that every optimization can be justified against real-user impact rather than adopted by default.
