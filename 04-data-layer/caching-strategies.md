# Caching Strategies

Caching is storing a copy of data closer to where it's needed to avoid redundant work. In frontend, there are four distinct cache layers: HTTP cache (browser), application cache (React Query), runtime cache (in-memory), and persistent cache (Service Worker). Each has different trade-offs around freshness, complexity, and control. The hardest problem in caching isn't storing data — it's knowing when to invalidate it.

---

## The Four Cache Layers

```
┌─────────────────────────────────────────────────────────────────┐
│                    Frontend Cache Layers                        │
│                                                                 │
│   Request travels inward until a cache hit is found:            │
│                                                                 │
│   Browser         App            Runtime        Server          │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐     │
│   │  HTTP    │   │  React   │   │  In-     │   │   CDN /  │     │
│   │  Cache   │──►│  Query   │──►│  Memory  │──►│  Origin  │     │
│   │          │   │  Cache   │   │  Map     │   │          │     │
│   └──────────┘   └──────────┘   └──────────┘   └──────────┘     │
│                                                                 │
│   Also: Service Worker cache sits between Browser and Network   │
└─────────────────────────────────────────────────────────────────┘
```

A well-designed system uses multiple layers. Each catches what the previous missed.

---

## Layer 1: HTTP Cache

The browser's built-in cache. Controlled by response headers from the server. Zero JavaScript required.

### Cache-Control Header

```
Cache-Control: public, max-age=31536000, immutable

public         → can be cached by browser AND intermediaries (CDNs, proxies)
private        → only the browser can cache (user-specific data)
max-age=N      → cache is valid for N seconds
immutable      → content will never change (safe to cache forever)
no-cache       → must revalidate with server before using cache
no-store       → never cache this response
```

### Strategies by Asset Type

```typescript
// Hashed JS/CSS bundles (main.a3f2c1.js)
// Content never changes for this URL → cache forever
Cache-Control: public, max-age=31536000, immutable

// HTML entry point (index.html)
// Must always be fresh → don't cache
// (if outdated, user loads old JS bundle references)
Cache-Control: no-cache

// API responses — depends on how often data changes
Cache-Control: public, max-age=60              // fresh for 60 seconds
Cache-Control: private, max-age=300           // user-specific, 5 minutes
Cache-Control: no-store                       // never cache (auth, payments)
```

### stale-while-revalidate

The most powerful HTTP cache strategy — serves stale data immediately while fetching fresh data in the background:

```
Cache-Control: max-age=60, stale-while-revalidate=300

Timeline:
  0–60s:    Cache is fresh → serve immediately, no network request
  60–360s:  Cache is stale → serve stale data immediately
             AND fetch fresh data in the background
  > 360s:   Cache is too stale → fetch fresh data, show loading
```

This is the same pattern React Query implements in JavaScript — but at the HTTP layer, for free.

### ETag: Conditional Requests

ETags let the browser revalidate without re-downloading the full response:

```
First request:
  GET /api/products
  ← 200 OK
  ← ETag: "abc123"
  ← Cache-Control: max-age=60
  ← [full response body: 50kb]

After 60s (cache expired), browser revalidates:
  GET /api/products
  → If-None-Match: "abc123"

If data hasn't changed:
  ← 304 Not Modified   (no body — saves the 50kb download)
  ← ETag: "abc123"

If data changed:
  ← 200 OK
  ← ETag: "xyz789"
  ← [new response body: 52kb]
```

ETags are especially valuable for large, infrequently changing responses.

---

## Layer 2: Application Cache (React Query)

React Query's in-memory cache is the primary application-level cache. It sits above HTTP and gives you programmatic control: invalidate on mutation, set TTLs per query, prefetch on hover.

The full React Query cache lifecycle is covered in `server-state-react-query.md`. Key invalidation patterns:

```typescript
const queryClient = useQueryClient();

// Invalidate all product queries after any product mutation
queryClient.invalidateQueries({ queryKey: ["products"] });

// Invalidate a specific product
queryClient.invalidateQueries({ queryKey: ["products", productId] });

// Directly update cache without network (after mutation returns the updated object)
queryClient.setQueryData(["products", productId], updatedProduct);

// Remove from cache entirely (forces fresh fetch on next access)
queryClient.removeQueries({ queryKey: ["products", productId] });
```

### Cache Warming

Populate the cache before the user asks for data:

```typescript
// On app startup — prefetch critical data
async function warmCache(queryClient: QueryClient) {
  await Promise.all([
    queryClient.prefetchQuery({
      queryKey: ["currentUser"],
      queryFn: authApi.getUser,
      staleTime: 300_000,
    }),
    queryClient.prefetchQuery({
      queryKey: ["featureFlags"],
      queryFn: configApi.getFlags,
      staleTime: Infinity, // flags don't change during a session
    }),
  ]);
}
```

---

## Layer 3: In-Memory Cache (Manual)

For data that doesn't need React Query's full machinery — utility functions, computed values, API responses outside React:

```typescript
// Simple Map-based cache with TTL
class MemoryCache<T> {
  private cache = new Map<string, { value: T; expiresAt: number }>();

  get(key: string): T | null {
    const entry = this.cache.get(key);
    if (!entry) return null;
    if (Date.now() > entry.expiresAt) {
      this.cache.delete(key);
      return null;
    }
    return entry.value;
  }

  set(key: string, value: T, ttlMs: number): void {
    this.cache.set(key, { value, expiresAt: Date.now() + ttlMs });
  }

  delete(key: string): void {
    this.cache.delete(key);
  }
}

// Usage — cache geocoding results to avoid duplicate API calls
const geocodeCache = new MemoryCache<Coordinates>();

async function geocode(address: string): Promise<Coordinates> {
  const cached = geocodeCache.get(address);
  if (cached) return cached;

  const result = await geocodingApi.lookup(address);
  geocodeCache.set(address, result, 1000 * 60 * 60); // 1 hour TTL
  return result;
}
```

---

## Layer 4: Service Worker Cache

Service Workers intercept network requests and can serve responses from a cache. Unlike HTTP cache, you control the caching logic in JavaScript — enabling offline support and fine-grained strategies.

```
┌─────────────────────────────────────────────────────────┐
│              Service Worker Request Flow                │
│                                                         │
│   App ──► Service Worker ──► Cache? ──► YES → respond   │
│                         │                               │
│                         └──► NO  ──► Network → Cache    │
│                                        response         │
└─────────────────────────────────────────────────────────┘
```

### Core Service Worker Strategies

**Cache First** — serve from cache, fall back to network. Best for static assets.

```javascript
// Good for: fonts, images, versioned JS/CSS
self.addEventListener("fetch", (event) => {
  event.respondWith(
    caches
      .match(event.request)
      .then((cached) => cached ?? fetch(event.request)),
  );
});
```

**Network First** — try network, fall back to cache. Best for API data.

```javascript
// Good for: API responses that must be fresh
self.addEventListener("fetch", (event) => {
  event.respondWith(
    fetch(event.request)
      .then((response) => {
        const clone = response.clone();
        caches
          .open("api-cache")
          .then((cache) => cache.put(event.request, clone));
        return response;
      })
      .catch(() => caches.match(event.request)), // offline fallback
  );
});
```

**Stale While Revalidate** — serve cache immediately, update in background.

```javascript
// Good for: non-critical content that should feel instant
self.addEventListener("fetch", (event) => {
  event.respondWith(
    caches.open("dynamic").then((cache) =>
      cache.match(event.request).then((cached) => {
        const fetchPromise = fetch(event.request).then((response) => {
          cache.put(event.request, response.clone());
          return response;
        });
        return cached ?? fetchPromise; // return cache immediately if available
      }),
    ),
  );
});
```

### Workbox (Recommended)

Building Service Workers manually is error-prone. Workbox is Google's library that implements these strategies with production-ready code:

```javascript
// vite.config.ts (using vite-plugin-pwa)
import { VitePWA } from "vite-plugin-pwa";

export default {
  plugins: [
    VitePWA({
      workbox: {
        runtimeCaching: [
          {
            urlPattern: /^https:\/\/api\.example\.com\/products/,
            handler: "NetworkFirst",
            options: {
              cacheName: "api-products",
              expiration: { maxAgeSeconds: 300 }, // 5 minutes
            },
          },
          {
            urlPattern: /\.(js|css|woff2)$/,
            handler: "CacheFirst",
            options: {
              cacheName: "static-assets",
              expiration: { maxAgeSeconds: 31536000 }, // 1 year
            },
          },
        ],
      },
    }),
  ],
};
```

---

## Cache Invalidation

"There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

### The Core Challenge

```
User loads product page → cached price: $129.99
Admin updates price to $99.99 on the backend
User sees $129.99 (stale) → places order expecting discount
User is confused → support ticket → bad experience
```

### Strategies

**Time-based (TTL)** — simplest, never perfectly fresh

```typescript
// Accept some staleness — refresh every 5 minutes
staleTime: 1000 * 60 * 5;
```

**Event-based** — invalidate when you know data changed

```typescript
// After updating a product, invalidate its cache
const updateProduct = useMutation({
  mutationFn: productsApi.update,
  onSuccess: (updated) => {
    queryClient.invalidateQueries({ queryKey: ["products"] });
  },
});
```

**WebSocket-driven** — server pushes invalidation signal

```typescript
// Server tells client "product p1 changed" → client refetches
const socket = new WebSocket("wss://api.example.com/events");

socket.addEventListener("message", (event) => {
  const { type, payload } = JSON.parse(event.data);

  if (type === "product:updated") {
    queryClient.invalidateQueries({ queryKey: ["products", payload.id] });
  }
});
```

**Optimistic invalidation** — update cache before server confirms (see server-state-react-query.md)

---

## Choosing a Strategy

```
Data type                    Strategy
─────────────────────────────────────────────────────────
Static assets (hashed URLs)  HTTP: Cache-Control: immutable
HTML entry point             HTTP: no-cache
User-specific API data       React Query: short staleTime + private
Public API data              React Query + HTTP: max-age + SWR
Reference data (currencies)  React Query: staleTime: Infinity
Offline-critical content     Service Worker: Cache First
Real-time data               WebSocket + invalidation, no long TTL
Auth tokens                  No cache — always validate
Payment data                 Cache-Control: no-store always
```
