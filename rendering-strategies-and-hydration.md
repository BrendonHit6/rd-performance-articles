# Rendering Strategies and Hydration

## Purpose

The second article in the series treats rendering models and hydration as a single topic. The rendering model determines when and where HTML is produced; the hydration strategy determines how that HTML becomes interactive. Both decisions directly affect the metrics established in the previous article: LCP (how quickly the main content appears) and INP (interface responsiveness).

## Base Rendering Models

Client-Side Rendering (CSR). The server returns minimal HTML and a JavaScript bundle; the page is built in the browser after scripts load and execute. It provides smooth interactivity but has a slower first paint and weaker baseline SEO, especially for content-heavy pages.

Server-Side Rendering (SSR). HTML is produced on the server per request and sent to the client ready-made. It improves time to first paint and indexing. Suitable for dynamic, frequently changing, or personalized content.

Static Site Generation (SSG). HTML is generated at build time and served as static files. It delivers the best performance and minimal server load. Suitable for content that changes rarely: blogs, documentation, marketing pages.

Incremental Static Regeneration (ISR). An evolution of SSG: static pages are regenerated on demand or on an interval, combining the performance of static output with data freshness.

```
WHERE / WHEN HTML IS PRODUCED
Model | Produced        | When          | First paint | Server load | Freshness
------+-----------------+---------------+-------------+-------------+----------
CSR   | Browser         | after JS runs | slow        | low         | live
SSR   | Server          | per request   | fast        | high        | live
SSG   | Build server    | at build time | fastest     | minimal     | stale
ISR   | Build + on-demand| build + revalidate | fastest | low       | near-live
```

The base models in Next.js map to explicit APIs:

```tsx
// SSR — rendered per request
export async function getServerSideProps() {
  const data = await fetch(`${API}/dashboard`).then(r => r.json())
  return { props: { data } }
}

// SSG — rendered at build time
export async function getStaticProps() {
  const posts = await fetch(`${API}/posts`).then(r => r.json())
  return { props: { posts } }
}

// ISR — SSG + periodic regeneration
export async function getStaticProps() {
  const posts = await fetch(`${API}/posts`).then(r => r.json())
  return { props: { posts }, revalidate: 60 } // regenerate at most once per 60s
}
```

In the App Router, the mode is locked per route, and ISR supports surgical invalidation by tag:

```tsx
// app/blog/[slug]/page.tsx — static with hourly ISR
export const dynamic = 'force-static'
export const revalidate = 60 * 60

// app/dashboard/page.tsx — always dynamic
export const dynamic = 'force-dynamic'
```

```tsx
// Invalidate a specific page on content change
'use server'
import { revalidateTag } from 'next/cache'

export async function revalidateProduct(id: string) {
  revalidateTag(`product:${id}`)
}
```

The practical principle is to choose the model per route, by data-freshness requirements, rather than applying one strategy to the entire application. Overusing dynamic rendering raises TTFB and server cost; an overly long revalidation interval yields stale content.

## Hydration as a Distinct Phase

Hydration is the process by which client-side JavaScript "brings to life" server-rendered or static HTML: it attaches event handlers, restores application state, and enables dynamic behavior. The browser first shows the ready HTML, then loads the bundle in the background and binds client logic to the markup.

```
HYDRATION TIMELINE (time ->)
| server HTML | download JS bundle | hydrate (main thread) | interactive |
      ^ visible          ^ still not          ^ attaching          ^ TTI / INP
        (fast paint)       interactive          handlers             ready
        <----------------- gap felt as "looks done but frozen" ----->
  goal: shrink the JS + hydration work to close this gap
```

A key caveat: hydration itself is a performance bottleneck. It runs on the main thread and delays the moment the page becomes truly interactive. Therefore the amount of client JavaScript and the hydration approach directly affect INP and TTI.

A separate class of problems is the hydration mismatch, where server and client markup diverge — for example, due to rendering time or random values, or accessing browser APIs during server rendering. This triggers a re-render and unnecessary CPU load. The correct approach is to isolate browser-dependent code and pass unstable values (time, state) as props rather than suppressing the warning:

```tsx
'use client'
import { useEffect, useState } from 'react'

// Gate browser-only rendering to avoid server/client divergence
export function ClientOnly({ children }: { children: React.ReactNode }) {
  const [ready, setReady] = useState(false)
  useEffect(() => setReady(true), [])
  return ready ? <>{children}</> : null
}
```

```tsx
// Pass canonical, server-computed time as a prop instead of
// calling Date.now()/Math.random() during render on both sides
const serverNow = new Date().toISOString()
return <ClientClock serverNow={serverNow} />
```

## Modern Hydration Strategies

Streaming SSR with Suspense. The server sends HTML in chunks: static and fast content appears immediately, while slow data-dependent blocks stream in progressively. This improves early paint (FCP/LCP) without blocking the page on the slowest request. Do not wrap the entire page in a single Suspense boundary — otherwise blocking persists.

```
STREAMING SSR WITH SUSPENSE
server ---> [Hero HTML] --------> shown instantly
       \
        \--> [Suspense: LatestDeals]
              (awaiting slow data)
              ...later... ---> [LatestDeals HTML] streamed in
  page is usable while the slow block is still loading
```

```tsx
// app/page.tsx
import { Suspense } from 'react'
import Hero from './_components/hero'
import LatestDeals from './_components/latest-deals' // async Server Component

export default function Page() {
  return (
    <>
      <Hero /> {/* instant */}
      <Suspense fallback={<div className="skeleton" />}>
        <LatestDeals /> {/* streams in later */}
      </Suspense>
    </>
  )
}
```

Selective / partial hydration. Only the interactive part of the page is hydrated, not the whole document. This reduces the amount of JavaScript and speeds up reaching interactivity.

Progressive hydration. A single application is hydrated in stages — by route boundaries, viewport visibility, user interaction, or idle time. The application remains a single unit, but its hydration is split into manageable parts.

Island architecture. The page consists mostly of static HTML with embedded independent interactive "islands." Each island has its own hydration boundary, loads its own bundle, and may even use a different framework. In Astro, the load timing is chosen per island:

```jsx
---
import Counter from './Counter.jsx'
---
<h1>Hello Astro</h1>
<Counter client:visible /> {/* hydrate only when scrolled into view */}
{/* other directives: client:load, client:idle */}
```

Resumability. A fundamentally different approach (implemented, for example, in Qwik): instead of re-executing logic on the client, state is restored directly from the HTML, and hydration as a distinct step is eliminated. JavaScript loads lazily — only what a specific interaction requires:

```tsx
import { component$, useStore } from '@builder.io/qwik'

export const Counter = component$(() => {
  const state = useStore({ count: 0 })
  // No hydration step — reactivates from serialized HTML state on interaction
  return <button onClick$={() => state.count++}>Count: {state.count}</button>
})
```

React Server Components. Components that run only on the server and do not ship their code to the client. They reduce bundle size and hydration load; client code is added only where state or effects are genuinely needed:

```tsx
// Server Component by default — no client JS shipped
export default async function ProductSpecs({ id }: { id: string }) {
  const specs = await getProductSpecs(id) // runs on the server
  return <table>{/* ... */}</table>
}

// Only the small interactive part is a client component
'use client'
export function QtyStepper({ onChange }: { onChange: (n: number) => void }) {
  // minimal client-side code
}
```

## Distinguishing the Concepts

Three approaches are often confused, so they should be distinguished explicitly. Progressive hydration optimizes the hydration of a single monolithic application. Island architecture builds the page as a set of small independent interactive applications within static HTML. Resumability abandons the hydration step altogether, restoring state from the markup. The goal in all three is the same — less JavaScript and faster interactivity — but the architectural decisions differ.

```
MODERN STRATEGIES COMPARED
                    | Unit           | Hydration     | Client JS
--------------------+----------------+---------------+-----------
Progressive hydration| one app (monolith)| staged      | high (full app)
Island architecture | many small apps| per-island    | low
Resumability        | one app        | none (resume) | minimal (lazy)
```

## Architectural Implications and Trade-offs

The choice of rendering model is a trade-off among data freshness, performance, and infrastructure cost. SSG/ISR minimize server load at the cost of more complex freshness management; SSR delivers fresh data at the cost of higher TTFB and server load; CSR simplifies infrastructure but degrades early paint and SEO.

Hydration strategies carry their own trade-offs. Island architecture and resumability yield minimal client JavaScript but impose architectural constraints — isolated state, no shared global context — and suit content sites better than complex applications. Progressive hydration preserves the full application model (routing, state management, forms) at the cost of a larger runtime. RSC reduce the bundle but add complexity in splitting server and client components.

Practical rule: content sites lean toward island architecture or resumability with minimal JavaScript; complex interactive applications (dashboards, CRMs, admin panels) lean toward SSR with progressive hydration and full framework tooling. These decisions should be fixed per route and controlled with the same performance budgets as in the previous article.

## Summary

The rendering model determines when HTML is produced; the hydration strategy determines the cost of its interactivity. Together they shape LCP and INP before any point optimizations. The next article covers caching at all levels — from HTTP and CDN to ISR and the data layer — as a way to lock in and scale the performance gains achieved here.
