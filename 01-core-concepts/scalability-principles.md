# Scalability Principles in Frontend

Frontend scalability isn't just about handling more users — it's about keeping the codebase, team, and user experience healthy as complexity grows. There are four dimensions: user scale, feature scale, team scale, and data scale. Each requires different strategies.

---

## The Four Dimensions of Frontend Scale

Most engineers think "scale" means traffic. In frontend, that's rarely the bottleneck. The real dimensions are:

```
┌─────────────────────────────────────────────────────────────┐
│                  Frontend Scale Dimensions                  │
│                                                             │
│   👤 User Scale       📦 Feature Scale                       │
│   More concurrent     More features,                        │
│   users, more         more routes,                          │
│   geographies         more complexity                       │
│                                                             │
│   👥 Team Scale       🗄️  Data Scale                         │
│   More engineers,     More items to                         │
│   more teams,         render, larger                        │
│   more coordination   API payloads                          │
└─────────────────────────────────────────────────────────────┘
```

A system design decision that helps one dimension can hurt another. Knowing which dimension matters most for your product is the first step.

---

## User Scale

When more users hit your frontend, the bottleneck is usually **delivery** — how fast assets reach the browser — not the application logic itself.

### CDN: The First Line of Defense

A Content Delivery Network serves your static assets (JS, CSS, images, fonts) from edge nodes close to the user. Without a CDN, every user downloads from your origin server.

```
Without CDN:
User (Istanbul) ──────────────────────► Origin (US East)
                    ~180ms latency

With CDN:
User (Istanbul) ──► Edge Node (Frankfurt) ──► Origin (US East)
                    ~12ms latency          (only on cache miss)
```

What to put on the CDN:

- JavaScript bundles (hashed filenames → long cache TTL)
- CSS files
- Images and fonts
- HTML files (for SSG / static sites)

What NOT to put on the CDN (without care):

- Personalized or user-specific content
- Frequently changing data without proper cache invalidation

### Cache Headers

```
# Immutable assets (hashed filename: main.a3f2c1.js)
Cache-Control: public, max-age=31536000, immutable

# HTML entry point (must revalidate to pick up new bundle references)
Cache-Control: no-cache

# API responses (short TTL, allow stale while revalidating)
Cache-Control: public, max-age=60, stale-while-revalidate=300
```

The pattern: **hash your assets, cache them forever, never cache your HTML.**

### Rendering Strategy by User Scale

Different rendering strategies handle user scale differently:

```
┌───────────────┬──────────────┬──────────────┬──────────────────┐
│               │     CSR      │     SSR      │       SSG        │
│               │ (Client-Side │ (Server-Side │ (Static Site     │
│               │  Rendering)  │  Rendering)  │   Generation)    │
├───────────────┼──────────────┼──────────────┼──────────────────┤
│ Server load   │ Low          │ High         │ Very low         │
│ on traffic    │              │              │                  │
├───────────────┼──────────────┼──────────────┼──────────────────┤
│ CDN-cacheable │ ✓ (shell)    │ ✗ (dynamic)  │ ✓ (fully)        │
├───────────────┼──────────────┼──────────────┼──────────────────┤
│ First load    │ Slow         │ Fast         │ Fastest          │
│ performance   │              │              │                  │
├───────────────┼──────────────┼──────────────┼──────────────────┤
│ Best for      │ Dashboards,  │ E-commerce,  │ Marketing sites, │
│               │ internal     │ news, SEO-   │ docs, blogs      │
│               │ tools        │ critical     │                  │
└───────────────┴──────────────┴──────────────┴──────────────────┘
```

> At very high user scale, SSR can become a bottleneck — every request hits your server. SSG + ISR (Incremental Static Regeneration) is often a better answer for content-heavy sites.

---

## Feature Scale

As the product grows, the frontend codebase accumulates features. Without intentional architecture, it becomes a **big ball of mud** — everything depends on everything else.

### Bundle Size Budget

Set a budget before you exceed it:

```
┌─────────────────────────────────────────┐
│          Bundle Size Budget             │
│                                         │
│  Initial JS:    < 200kb (gzipped)       │
│  Per-route JS:  < 50kb  (gzipped)       │
│  Total images:  < 500kb (above fold)    │
│  Web Fonts:     < 100kb                 │
└─────────────────────────────────────────┘
```

These aren't universal rules — they're starting points. The key is having a budget at all, and enforcing it in CI.

```json
// package.json — fail the build if budget is exceeded
{
  "bundlesize": [{ "path": "./dist/main.*.js", "maxSize": "200 kB" }]
}
```

### Code Splitting as a Scaling Strategy

Without code splitting, every new feature adds to the initial bundle. With it, each feature pays its own cost only when loaded.

```
Without splitting:
┌──────────────────────────────────────────────────┐
│ Auth + Products + Cart + Checkout + Admin + ...  │  → 1.2MB initial
└──────────────────────────────────────────────────┘

With route-based splitting:
┌────────────┐  ┌────────────┐  ┌─────────────┐
│    Auth    │  │  Products  │  │  Checkout   │  → 180KB initial
│   (core)   │  │  (lazy)    │  │  (lazy)     │
└────────────┘  └────────────┘  └─────────────┘
                  loads only        loads only
                  on /products      on /checkout
```

The initial bundle should only contain what every user needs on every page: auth, layout, routing.

### Dependency Audit

Third-party libraries are the silent killers of bundle size:

```bash
# Analyze what's in your bundle
npx vite-bundle-visualizer     # Vite
npx webpack-bundle-analyzer    # Webpack
```

Common offenders and alternatives:

| Library            | Size  | Alternative                 | Size   |
| ------------------ | ----- | --------------------------- | ------ |
| `moment.js`        | 67kb  | `date-fns` (tree-shakeable) | ~3kb   |
| `lodash`           | 72kb  | `lodash-es` + tree shaking  | ~1–5kb |
| `axios`            | 13kb  | `fetch` (native)            | 0kb    |
| Full `react-icons` | 40MB+ | Import individually         | ~1kb   |

> **Rule:** Before adding a dependency, check its size on [bundlephobia.com](https://bundlephobia.com). Every kilobyte has a cost.

---

## Team Scale

This is the most underestimated dimension. An app with 3 engineers can get away with anything. An app with 20 engineers on 4 teams needs explicit contracts between modules.

### Coupling is the Enemy

Coupling means: changing one thing requires changing another. It compounds as the team grows.

```
Low coupling (good):
Team A changes CartFeature ──► No impact on ProductsFeature
Team B changes ProductsFeature ──► No impact on CartFeature

High coupling (bad):
Team A changes CartFeature ──► Breaks ProductsFeature
                            ──► Breaks CheckoutFeature
                            ──► Breaks 3 shared components
```

Strategies to reduce coupling:

**1. Enforce module boundaries** (see monolith-vs-modular.md)
Each feature exports only through `index.ts`. No reaching into internals.

**2. Own your data**
Each feature owns its API calls and state. Don't share store slices between unrelated features.

**3. Use events or callbacks for cross-feature communication**
Features don't import each other — they communicate through a shared event bus or callbacks from the shell.

### Shared vs Feature-owned Code

A common mistake: putting too much in `shared/`. Over time, `shared/` becomes a dumping ground that everyone depends on and nobody owns.

```
Rule of three: a piece of code should be duplicated twice
before it's moved to shared/. Premature abstraction creates
more coupling than duplication.
```

```
shared/              ← only things used by 3+ features
├── ui/              ← design system (Button, Input, Modal)
├── lib/             ← API client, date utils, validators
└── types/           ← shared TypeScript interfaces

features/cart/       ← cart-specific utils stay here
└── lib/
    └── formatPrice.ts   ← not shared until 2 other features need it
```

### Conventions as Scale

As teams grow, conventions matter more than tools. Document these explicitly:

```markdown
## Team Conventions

- All API calls live in `feature/api/` — never in components
- Components never call fetch() directly — always through hooks
- Global state only for: auth, theme, notifications
- Local state for everything else unless proven otherwise
- Every PR must include tests for new business logic
```

---

## Data Scale

When the UI needs to render or manage large amounts of data, different strategies apply.

### Pagination vs Infinite Scroll vs Virtualization

```
┌─────────────────┬────────────────────┬───────────────────────┐
│   Pagination    │  Infinite Scroll   │    Virtualization     │
├─────────────────┼────────────────────┼───────────────────────┤
│ Load page N     │ Load more on       │ Render only visible   │
│ on demand       │ scroll to bottom   │ rows (all data local) │
├─────────────────┼────────────────────┼───────────────────────┤
│ Good for:       │ Good for:          │ Good for:             │
│ Search results  │ Social feeds       │ Data grids, logs,     │
│ tables, grids   │ product listings   │ large local datasets  │
├─────────────────┼────────────────────┼───────────────────────┤
│ DOM nodes: low  │ DOM nodes: grows   │ DOM nodes: constant   │
│ UX: interruptive│ UX: seamless       │ UX: seamless          │
└─────────────────┴────────────────────┴───────────────────────┘
```

Virtualization renders only the rows currently in the viewport — a list of 100,000 items maintains the same DOM footprint as a list of 20:

```tsx
import { useVirtualizer } from "@tanstack/react-virtual";

function VirtualList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48, // estimated row height in px
  });

  return (
    <div ref={parentRef} style={{ height: "600px", overflow: "auto" }}>
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: "relative",
        }}
      >
        {virtualizer.getVirtualItems().map((virtualRow) => (
          <div
            key={virtualRow.index}
            style={{
              position: "absolute",
              top: `${virtualRow.start}px`,
              width: "100%",
            }}
          >
            {items[virtualRow.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Normalizing API Responses

Large, nested API responses are expensive to traverse and update. Normalize them into flat structures:

```typescript
// Raw API response — deeply nested, hard to update
{
  "order": {
    "id": "o1",
    "items": [
      { "id": "i1", "product": { "id": "p1", "name": "Keyboard" } },
      { "id": "i2", "product": { "id": "p2", "name": "Mouse" } }
    ]
  }
}

// Normalized — flat, easy to update, no duplication
{
  orders: { "o1": { id: "o1", itemIds: ["i1", "i2"] } },
  items:  { "i1": { id: "i1", productId: "p1" },
            "i2": { id: "i2", productId: "p2" } },
  products: { "p1": { id: "p1", name: "Keyboard" },
              "p2": { id: "p2", name: "Mouse" } }
}
```

Libraries like `normalizr` or RTK Query's built-in normalization handle this automatically.

---

## Putting It Together: Scalability Checklist

Use this when evaluating or designing a frontend system:

```
User Scale
□ Static assets served from CDN?
□ Correct cache headers on assets vs HTML?
□ Rendering strategy matches content type (SSG/SSR/CSR)?
□ Images optimized and served in modern formats (WebP, AVIF)?

Feature Scale
□ Bundle size budget defined and enforced in CI?
□ Route-based code splitting in place?
□ Third-party dependencies audited for size?

Team Scale
□ Feature boundaries enforced (index.ts barrels)?
□ Shared/ directory kept lean?
□ Team conventions documented?
□ ESLint rules prevent cross-feature internal imports?

Data Scale
□ Large lists use virtualization?
□ Pagination or infinite scroll for server-side data?
□ API responses normalized if deeply nested?
```
