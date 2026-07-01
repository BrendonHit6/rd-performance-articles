# Core Web Vitals: A Coordinate System for Performance Optimization

## Purpose

This article opens a series on frontend performance. Its goal is to establish a common vocabulary, target thresholds, and optimization practices that subsequent topics will build on: rendering strategies, caching, and bundle analysis. Core Web Vitals are treated here not as an end in themselves, but as a coordinate system — a set of metrics against which the effect of any optimization is measured.

## The Three Metrics

Core Web Vitals cover three aspects of user experience: loading speed, interface responsiveness, and visual stability.

Largest Contentful Paint (LCP) measures the time until the largest visible element on the page — usually an image or text block — becomes visible. It reflects perceived loading speed. Target threshold: <= 2.5 s.

Interaction to Next Paint (INP) measures the page's responsiveness to user interactions — from action to visual feedback. Target threshold: <= 200 ms. Note: INP replaced the earlier First Input Delay (FID) metric. Some existing guides still reference FID; responsiveness should be assessed via INP, as it accounts for all interactions during a session rather than only the first.

Cumulative Layout Shift (CLS) measures visual stability — the total amount of unexpected content shifts after the initial render. Target threshold: <= 0.1.

Thresholds are considered met when the corresponding value holds for at least 75% of real-user visits.

```
CORE WEB VITALS
+-----------------------------------------------------------+
| Metric | Measures            | Good    | Needs improvement |
+--------+---------------------+---------+-------------------+
| LCP    | Loading (main       | <=2.5 s | 2.5 s - 4.0 s     |
|        | element visible)    |         |                   |
| INP    | Responsiveness      | <=200ms | 200 ms - 500 ms   |
|        | (action -> paint)   |         |                   |
| CLS    | Visual stability    | <=0.1   | 0.1 - 0.25        |
|        | (unexpected shifts) |         |                   |
+--------+---------------------+---------+-------------------+
  Thresholds must hold for >=75% of real-user visits (p75)
```

## Lab Data and Field Data

Metrics are collected in two ways, and the two must not be conflated.

Lab data are synthetic measurements taken in a controlled environment (for example, a Lighthouse audit). They are reproducible and convenient for debugging, but run on fixed hardware and network conditions.

Field data are real-user measurements (Real User Monitoring). They reflect actual experience across different devices, networks, and geographies. It is field data that determines compliance with thresholds at the 75th percentile.

Practical takeaway: a lab result is a guideline for development, but the criterion of success remains field data.

```
LAB (synthetic)              FIELD (real users / RUM)
+-------------------+        +----------------------------+
| Lighthouse /      |        | Real devices, networks,    |
| PageSpeed         |        | geographies                |
| fixed HW + net    |        | collected in production    |
+---------+---------+        +--------------+-------------+
          |                                 |
          v                                 v
   guides development              source of truth (p75)
          \\                             /
           \\____ measure before/after _/
                  track the trend
```

## Causes of Degradation

LCP suffers from large JavaScript bundles that block rendering, unoptimized images, slow server response, and render-blocking resources. INP is degraded by long tasks on the main thread, excessive re-renders, and heavy computation during interaction handling. CLS increases due to images without specified dimensions, late-loading dynamic content, and text resizing during font loading.

## Optimization Practices

### LCP: Shorten the Path to Main Content

Code splitting and lazy loading. Splitting code by route reduces the initial bundle: the browser loads only what the current page needs. Components below the fold load on demand. The key caveat — do not apply lazy loading to above-the-fold content that forms the LCP.

Critical path optimization. Critical CSS for above-the-fold content is inlined so styles apply without waiting for external stylesheets. Critical fonts and the likely LCP image are given preload for immediate download.

Image optimization. Modern formats (WebP, AVIF) provide better compression. Responsive loading ensures a device receives appropriately sized images. The LCP image is marked as priority; blur-up placeholders show an interim result before the full version loads.

Server-side rendering and streaming. SSR sends pre-rendered HTML, so the user sees content without waiting for JavaScript execution. Streaming with Suspense extends the approach: fast content renders immediately, slow content arrives progressively.

```
LCP CRITICAL PATH (time ->)
0ms                                                   2500ms
|----------|-----------|--------------|-----------------|
 TTFB       CSS/fonts    JS parse/exec   LCP element
 (server)   (blocking)   (main thread)   painted
   ^ SSR/streaming   ^ inline critical   ^ preload LCP
     shortens TTFB     CSS, preload         image / priority
                       fonts
```

### INP: Free the Main Thread During Interactions

Debouncing. Expensive operations do not run on every keystroke or event — processing starts after a pause in input.

React transitions. Updates unrelated to direct interaction are marked as low priority, so urgent updates (user input) are handled first and the interface stays responsive.

Virtualizing long lists. Only the visible portion of a list is rendered, sharply reducing DOM size and reconciliation work for large datasets.

Memoization. Caching expensive computations and skipping re-renders of components with unchanged props reduces unnecessary work.

Web Workers. Heavy computation (parsing, data processing) is moved to a separate thread, leaving the main thread free for rendering and interactions. Breaking up long tasks with periodic yielding to the browser allows pending interactions to be handled.

```
INP: MAIN THREAD DURING AN INTERACTION
 click
   |
   v
 [====== long task 320ms ======]  ......  [paint]
   ^ input delayed; page feels unresponsive

 AFTER: yield + offload
 click
   |
   v
 [task 40ms] yield [task 40ms] yield [paint]   <=200ms
   ^ break long tasks, move heavy work to Web Worker
```

### CLS: Fix the Layout

Reserving space. Dimensions are set for images and dynamic content (width/height or aspect-ratio) so that loading does not shift the layout.

Font loading strategy. font-display: swap shows a fallback font immediately and swaps in the web font once loaded. Matching fallback font metrics to the web font minimizes the shift during the swap.

Skeleton screens. Placeholders with the dimensions of the final content prevent shift better than generic spinners: real content replaces them without altering the layout.

Avoiding content injection. Banners and notifications are not inserted in a way that shifts existing content; space is reserved for them or an overlay is used. CSS containment isolates the layout of individual elements, preventing internal changes from affecting the rest of the page.

```
NO RESERVED SPACE (bad)          RESERVED SPACE (good)
+----------------------+         +----------------------+
| Heading              |         | Heading              |
| [text text text]     |         | +------------------+ |
+----------------------+         | |  image (w x h)   | |
        | image loads            | +------------------+ |
        v  <-- content jumps     | [text text text]     |
+----------------------+         +----------------------+
| +------------------+ |            ^ box reserved via
| |  image (loaded)  | |              width/height /
| +------------------+ |              aspect-ratio =>
| [text pushed down]   |              CLS ~ 0
+----------------------+
```

### Cross-Cutting Quick Wins

A number of measures deliver significant effect for minimal effort: server-level compression (Brotli or Gzip); a CDN for static assets; adding dimensions to all images (eliminates most image-related CLS); lazy loading off-screen images; code splitting by route; preloading critical fonts; font-display: swap; minimizing third-party scripts; HTTP/2 or HTTP/3; and correct cache headers for repeat visits.

## Measurement

The minimum toolkit includes Lighthouse or PageSpeed Insights for lab measurements, the Performance panel in DevTools for debugging, and a mechanism for collecting field metrics. The working principle is to measure before and after each change and track the trend of field data rather than a one-off result.

## Architectural Implications and Trade-offs

The choice of measurement approach is an architectural decision. Synthetic monitoring is cheaper and deterministic but underestimates the problems of slow devices and networks; Real User Monitoring provides an accurate picture at the cost of additional instrumentation. The two approaches complement each other: synthetics during development, field data as the source of truth in production.

Optimization practices also carry trade-offs. Aggressive code splitting reduces the initial bundle but increases the number of network requests. Offloading computation to Web Workers frees the main thread at the cost of more complex inter-thread communication. SSR and streaming improve LCP but raise server load and infrastructure complexity. To control these trade-offs systematically, thresholds are enforced as performance budgets on bundle size and metric values, integrated into CI/CD; this moves performance from the status of a one-time optimization to a standing build requirement.

## Summary

Core Web Vitals define measurable acceptance criteria for the entire series: LCP <= 2.5 s, INP <= 200 ms, CLS <= 0.1 by field data. The next article covers rendering strategies (SSR/SSG and hydration) and shows how the choice of rendering model directly affects LCP and INP.
