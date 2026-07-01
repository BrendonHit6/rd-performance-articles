# Caching at All Levels

## Purpose

The third article in the series treats caching as a multi-layered architecture rather than a set of disconnected tricks. Caching is the primary lever for TTFB and LCP once a rendering model has been chosen. The article walks through every level — from the browser to the data layer — and shows how they cooperate under a single principle: serve something useful instantly, verify freshness in the background, update without drama.

## The Mental Model: Three Layers, One Contract

Data and assets flow through three cache layers, each closer to the user than the last:

```
        Request
           |
           v
 +-------------------+   fastest, per-user
 |   BROWSER LAYER   |   HTTP cache . memory . IndexedDB . data-layer
 +---------+---------+
           | miss
           v
 +-------------------+   shared, geo-distributed
 |    EDGE LAYER     |   CDN PoPs . edge functions . header shaping
 +---------+---------+
           | miss
           v
 +-------------------+   source of truth
 |   ORIGIN LAYER    |   server . data-layer cache (Redis) . database
 +-------------------+
```

The contract between layers is simple and repeatable: serve something immediately, verify in the background, update quietly. This is the stale-while-revalidate principle, applicable at every layer:

```
 t0 -- serve cached (stale) --> user sees content instantly
  |
  +-- revalidate in background --> update cache --> next read is fresh
```

## Layer 1: Browser Cache

The browser cache is an expiration-based cache driven by Cache-Control headers. Every cacheable object receives a time-to-live (TTL) via max-age; when an ETag or Last-Modified is present, the browser sends a conditional request and receives 304 Not Modified instead of re-downloading, saving bandwidth.

The key rule is to distinguish immutable from mutable resources. Content-hashed static assets are cached for as long as possible; frequently changing HTML is not cached or has a short TTL.

```
# Immutable, content-hashed assets (e.g., /assets/app.8f1a2.js)
Cache-Control: public, max-age=31536000, immutable

# HTML/routes that update often
Cache-Control: no-store
```

For JSON/API responses that must be both fast and fresh, apply SWR:

```
Cache-Control: public, max-age=0, s-maxage=60, stale-while-revalidate=30
ETag: "abc123"
```

Here `stale-while-revalidate` serves a stale response immediately while refreshing it in the background. A critical requirement is asset hashing: a content hash in the filename ensures that changing a resource changes its name, so immutable files can be cached safely and indefinitely. This eliminates the classic problem of users getting stuck on stale JavaScript.

## Layer 1a: The Client-Side Data Layer

Even with correct HTTP headers, UI state needs separate management. The application-level SWR pattern returns cached data instantly, fetches fresh data in the background, and re-renders:

```tsx
// useProducts.ts
import useSWR from 'swr'

const fetcher = (url: string) => fetch(url).then(r => r.json())

export function useProducts() {
  const { data } = useSWR('/api/products', fetcher, {
    revalidateOnFocus: true,
    dedupingInterval: 10_000,
  })
  return data ?? []
}
```

This yields an instant paint from memory and correctness a moment later — a significant gain in perceived speed.

## Layer 2: CDN and Transport

A CDN is a globally distributed cache infrastructure: a user connects to the nearest edge location via Geo-DNS or anycast and receives cached resources from there, sharply reducing latency. The edge also terminates TLS near the client, shortening the handshake time.

```
   User (Tokyo)          User (Berlin)          User (Sao Paulo)
        |                     |                       |
        v                     v                       v
   [Edge: Tokyo]        [Edge: Frankfurt]       [Edge: Sao Paulo]
        +-------------------- miss ----------------------+
                              |
                              v
                      [ Origin server ]
```

At the transport level, HTTP/2 removes HTTP/1.1 limitations through multiplexing — parallel loading of resources over a single connection, header compression, and server push. This is especially noticeable on pages with many assets.

At the CDN, the same SWR applies, but via `s-maxage`, which controls the shared (edge) cache specifically:

```
Cache-Control: public, max-age=0, s-maxage=120, stale-while-revalidate=60
```

Edge functions normalize URLs, strip volatile headers, and collapse cache keys to avoid cache fragmentation.

## Layer 3: Service Workers

A service worker provides offline access, instant navigations, and "late but right" updates. Typical uses are app-shell caching, runtime caching with SWR, and background sync.

```js
// sw.js — stale-while-revalidate for /api/*
self.addEventListener('fetch', event => {
  const { request } = event
  if (request.url.includes('/api/')) {
    event.respondWith((async () => {
      const cache = await caches.open('api-v1')
      const cached = await cache.match(request)
      // Background refresh that does not block the UI
      const networkPromise = fetch(request).then(resp => {
        cache.put(request, resp.clone())
        return resp
      }).catch(() => cached) // fallback when offline
      // Return cache immediately, network when it resolves
      return cached || networkPromise
    })())
  }
})
```

It is important to version service-worker cache names on deploy (`api-v1` -> `api-v2`) so old caches drain correctly and do not conflict with the HTTP cache.

## Layer 4: Origin and the Data Layer

On the server side, caching addresses database load during traffic spikes. The most common pattern is **cache-aside** (lazy loading): the application reads the cache first; on a miss it queries the database, writes the result to the cache, and returns it.

```
        [ Read request ]
               |
               v
        Check cache store
               |
        +------+------+
      (hit)         (miss)
        |             |
        v             v
   Return data   Query database
                      |
                      v
                 Write to cache
                      |
                      v
                 Return data
```

The pattern is resilient (if the cache fails, the application works directly against the database) and memory-efficient, but has weaknesses: first-request latency, the risk of stale data, and cache stampede — where, on a popular key's TTL expiry, thousands of requests hit the database simultaneously.

An alternative for strong consistency is **write-through**: the write passes through the cache to the database synchronously, so the cache is never stale, at the cost of higher write latency.

```
[ Write ] --> [ Application ] --> [ Cache ] --> [ Database ]
                                     +-- both must succeed --+
```

In distributed applications (multiple instances), the framework's built-in cache is kept per instance, which breaks consistency. The solution is an external shared cache (for example, Redis) as a shared cache handler, with a fallback to a local LRU when it is unavailable:

```js
// cache-handler.mjs (simplified)
import createLruHandler from '@neshca/cache-handler/local-lru'
import createRedisHandler from '@neshca/cache-handler/redis-strings'
import { createClient } from 'redis'
import { CacheHandler } from '@neshca/cache-handler'

CacheHandler.onCreation(async () => {
  const client = createClient({ url: process.env.REDIS_URL })
  await client.connect().catch(() => {})
  const handler = client?.isReady
    ? await createRedisHandler({ client, keyPrefix: 'app_cache:' })
    : createLruHandler() // fallback
  return { handlers: [handler] }
})

export default CacheHandler
```

## The Hardest Problem: Invalidation

Cache invalidation is the hardest part of the topic: when data changes in the source of truth, the cache instantly becomes a source of error. Three main strategies apply.

**TTL** is the simplest defense: a key receives an absolute lifespan. Suitable for non-critical data (analytics, article content), where a few minutes of sync delay is acceptable.

**Explicit deletion** — the application actively evicts the stale key immediately after a database mutation:

```python
def update_product_price(product_id, new_price):
    db.execute("UPDATE products SET price = ? WHERE id = ?", (new_price, product_id))
    cache.delete(f"product:details:{product_id}")  # evict stale entry
```

Caveat: in a high-load environment, a race is possible where a read re-caches the old value between the update and the deletion.

**Event-driven (asynchronous) invalidation** — for large microservice systems: Change Data Capture tools or events on a stream (for example, Kafka) let independent workers purge the cache instantly without interfering with the user-facing write loop.

```
[ DB commit ] --> [ CDC / Kafka event ] --> [ Eviction worker ] --> [ Cache purge ]
```

At the rendering level, the same approach corresponds to surgical ISR revalidation by tag:

```tsx
'use server'
import { revalidateTag } from 'next/cache'

export async function onCatalogChanged() {
  revalidateTag('catalog') // regenerate only the related pages
}
```

## Choosing a Strategy

```
              What is the primary load?
                        |
        +---------------+---------------+
   Read-heavy                       Write-heavy
        |                               |
  Stale reads OK?               Strong consistency?
   +----+----+                   +------+------+
 (yes)     (no)                (yes)         (no)
   |         |                   |             |
   v         v                   v             v
Cache-Aside  Cache-Aside      Write-Through  Event-Driven
+ TTL        + Explicit                      Async Cache
             Invalidation
```

Typical mappings: product catalogs and listings -> cache-aside + TTL; user profiles and sessions -> write-through; financial ledgers -> minimal caching with database-level guarantees; real-time dashboards -> event-driven invalidation.

## Architectural Implications and Trade-offs

The choice of caching strategy is a trade-off among freshness, consistency, and cost. For read-heavy workloads with tolerable staleness, cache-aside with TTL is optimal; for read-heavy with strict requirements, cache-aside with explicit invalidation; for write-heavy with strict consistency, write-through; for large decoupled systems, event-driven invalidation. Financial operations require minimal caching and transactional guarantees at the database layer.

Each layer has its own risks that should be controlled systematically. Un-hashed assets with long TTLs "lock" users onto an old version. Overly aggressive HTML caching breaks personalization. Double caching (SW + HTTP) can conflict without versioning. Cache-key explosion from varying by unstable parameters fragments the cache and lowers the hit rate. The practical rule is to fix the cache policy as part of the architecture: asset hashing, cache-key normalization at the edge, versioning of SW caches on deploy, and explicit invalidation of changed routes. These decisions are controlled with the same performance budgets as in the previous articles.

## Summary

Caching is not a standalone trick but the way modern frontends work: browser, edge, and application logic all play by the shared stale-while-revalidate rule. Correctly built cache layers lock in the TTFB and LCP gains achieved by the rendering choice. The next article covers bundle analysis and code splitting — managing the amount of JavaScript, which directly affects INP.
