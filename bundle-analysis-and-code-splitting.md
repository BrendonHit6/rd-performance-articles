# Bundle Analysis and Code Splitting

## Introduction

As applications grow, the JavaScript bundle becomes one of the primary factors affecting load performance. Every kilobyte of JavaScript must be downloaded, parsed, compiled, and executed before the interface becomes interactive. Bundle analysis and code splitting are the practices that keep this cost under control. This article covers advanced techniques for measuring, reducing, and structuring bundles, together with the architectural trade-offs each decision introduces.

## Why Bundle Size Matters

JavaScript is the most expensive resource per byte. Unlike images, which the browser can decode off the main thread, JavaScript blocks the main thread during parsing and execution. A large bundle directly degrades Time to Interactive (TTI) and Interaction to Next Paint (INP), and inflates Largest Contentful Paint (LCP) when render-blocking scripts sit in the critical path.

The goal is not simply "smaller" but "less on the critical path": ship only the code required for the current view, and defer everything else.

## Bundle Analysis

Before optimizing, measure. Bundle analysis reveals what actually ends up in the output, which dependencies dominate, and where duplication occurs.

### Visualizing the Bundle

A treemap is the most effective way to understand composition. With webpack:

```js
// webpack.config.js
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      reportFilename: 'bundle-report.html',
      openAnalyzer: false,
    }),
  ],
};
```

For Next.js:

```js
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

module.exports = withBundleAnalyzer({});
```

```bash
ANALYZE=true npm run build
```

### Reading a Treemap

A treemap shows each module as a rectangle sized proportionally to its footprint. Large, unexpected blocks are the first optimization targets.

```
+-----------------------------------------------------+
|  main.js  (420 KB)                                  |
|  +---------------------------+  +----------------+  |
|  |  moment (68 KB)           |  |  lodash (24KB) |  |
|  |  [+ all locales]          |  +----------------+  |
|  +---------------------------+  +----------------+  |
|  +-------------------+  +-------+  | app code     |  |
|  |  chart-lib (92KB) |  | rxjs  |  | (110 KB)     |  |
|  +-------------------+  +-------+  +----------------+ |
+-----------------------------------------------------+
        ^ heavy 3rd-party deps dominate the bundle
```

Common findings: a date library shipping every locale, a utility library imported in full, duplicate copies of the same dependency at different versions, or a charting library loaded on pages that never render a chart.

## Code Splitting

Code splitting breaks the bundle into smaller chunks loaded on demand, so users download only what the current route or interaction requires.

### Route-Based Splitting

The highest-impact split is per route. With React:

```jsx
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

const Dashboard = lazy(() => import('./pages/Dashboard'));
const Reports = lazy(() => import('./pages/Reports'));
const Settings = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/reports" element={<Reports />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

### Component-Level and On-Demand Splitting

Heavy components that are not needed on first paint can be deferred until interaction.

```jsx
import dynamic from 'next/dynamic';

// Loaded only when rendered, and never on the server.
const HeavyChart = dynamic(() => import('../components/HeavyChart'), {
  ssr: false,
  loading: () => <p>Loading chart…</p>,
});
```

Large dependencies can be imported lazily at the moment they are used:

```js
async function runSearch(query, documents) {
  const Fuse = (await import('fuse.js')).default;
  const fuse = new Fuse(documents, { keys: ['title', 'body'] });
  return fuse.search(query);
}
```

## Tree Shaking

Tree shaking removes code that is never used from the final bundle. It relies on ES module static analysis: the bundler can determine which exports are referenced and drop the rest.

### Import Narrowly

```js
// Bad: pulls the entire library into the bundle
import _ from 'lodash';
_.debounce(fn, 200);

// Good: allows unused code to be dropped
import debounce from 'lodash/debounce';
debounce(fn, 200);
```

### Mark the Package as Side-Effect Free

Tree shaking only works when the bundler can prove that removing a module has no side effects.

```json
{
  "name": "my-lib",
  "sideEffects": false
}
```

If some files do have side effects (for example, global CSS imports), list them explicitly:

```json
{
  "sideEffects": ["*.css", "./src/polyfills.js"]
}
```

### Effect on the Bundle

```
BEFORE tree shaking          AFTER tree shaking
+---------------------+      +---------------------+
| lodash (full) 24 KB |      | debounce      1.2KB |
| moment (all) 68 KB  |  =>  | date-fns/fmt  3.0KB |
| used app code 40 KB |      | used app code 40 KB |
+---------------------+      +---------------------+
   total: 132 KB               total: ~44 KB
```

## Additional Techniques

Prefer smaller, modular dependencies (for example, date-fns over moment; native Intl over heavy formatting libraries). Deduplicate dependencies by aligning versions so the bundler does not include multiple copies. Extract a shared vendor chunk for stable third-party code so it can be cached across deployments. Compress output with Brotli or gzip at the server or CDN. Use modern output targets to avoid shipping unnecessary transpilation and polyfills to browsers that do not need them.

## Architectural Consequences and Trade-Offs

Code splitting is not free. Each split introduces an additional network request and a potential loading state, which can hurt perceived performance if applied too granularly. Over-splitting produces waterfalls of small chunks, each with its own latency; under-splitting ships dead weight on first load. The correct granularity is usually the route, with selective component-level splits for genuinely heavy, non-critical UI.

Lazy loading also shifts complexity into the runtime: fallbacks, error boundaries for failed chunk loads, and prefetching strategy (for example, prefetching the next likely route on hover or idle) all become part of the design. Setting `ssr: false` improves the initial bundle but removes that content from server-rendered HTML, with implications for SEO and first paint.

The mechanism that keeps all of this under control is the performance budget enforced in CI/CD. Bundle size limits should be treated as a build gate: a pull request that pushes a chunk over its threshold fails the pipeline, just like a failing test.

```json
// Example budget check with a size-limit style config
{
  "size-limit": [
    { "path": "dist/main.*.js", "limit": "170 KB" },
    { "path": "dist/vendor.*.js", "limit": "120 KB" }
  ]
}
```

This ties the entire series together: Web Vitals define the user-facing targets, rendering strategy decides where work happens, caching controls what is recomputed and re-fetched, and bundle discipline controls what is shipped in the first place. Enforced continuously in CI/CD, performance budgets turn these individual optimizations into a durable, regression-resistant system.

## Conclusion

Bundle analysis tells you where the weight is; code splitting and tree shaking remove it from the critical path. Applied deliberately — measured with a treemap, split at the route level, kept honest with a CI budget — these techniques deliver fast initial loads without sacrificing the richness of the application. This concludes the series on advanced front-end performance optimization: measurement, rendering, caching, and bundling, unified by performance budgets as the architectural control plane.
