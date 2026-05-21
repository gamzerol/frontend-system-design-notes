# Microfrontend Architecture

Microfrontends extend microservices thinking to the frontend — splitting a large application into independently developed, built, and deployed units. Each team owns their slice end-to-end. It solves organizational problems at large scale, but introduces real complexity. Most applications should not use microfrontends. When they should, Module Federation is the dominant modern approach.

---

## What Problem Does It Solve?

Microfrontends are an organizational solution, not a technical one. The pain they address is specific:

```
The Monolith Problem at Team Scale:

┌─────────────────────────────────────────────────┐
│              One Giant Frontend Repo            │
│                                                 │
│  Team A (Search) ──┐                            │
│  Team B (Catalog) ─┼──► Same codebase           │
│  Team C (Cart)   ──┤    Same build pipeline     │
│  Team D (Account)──┘    Same deployment         │
│                                                 │
│  Problems:                                      │
│  • Team A's bad deploy takes down everyone      │
│  • Team B can't release until Team C's bug fix  │
│  • Merge conflicts across team boundaries       │
│  • One slow test suite blocks all deployments   │
└─────────────────────────────────────────────────┘
```

If your team is smaller than ~20 engineers, this pain probably isn't bad enough to justify microfrontends. A well-organized monolith (feature-based architecture) solves most of the same problems with far less complexity.

---

## The Microfrontend Model

Each team owns a vertical slice of the product — from UI to backend service to database.

```
┌─────────────────────────────────────────────────────────────┐
│                      Shell / Host App                       │
│              (routing, auth, shared layout)                 │
│                    Owned by Platform Team                   │
└──────────┬──────────────────┬──────────────────┬────────────┘
           │                  │                  │
     loads │            loads │            loads │
     at runtime         at runtime         at runtime
           │                  │                  │
┌──────────▼──────┐  ┌────────▼────────┐  ┌─────▼──────────┐
│   Search MFE    │  │   Catalog MFE   │  │   Cart MFE     │
│                 │  │                 │  │                │
│  React 18       │  │  React 18       │  │  Vue 3         │
│  Own CI/CD      │  │  Own CI/CD      │  │  Own CI/CD     │
│  Own deploy     │  │  Own deploy     │  │  Own deploy    │
│  Team A owns    │  │  Team B owns    │  │  Team C owns   │
└─────────────────┘  └─────────────────┘  └────────────────┘
```

Key properties:

- Each MFE is a **separate repository** with its own build pipeline
- Each MFE can be deployed **independently** — Team A ships without waiting for Team B
- Each MFE can use **different frameworks** (though this is usually avoided)
- The shell composes them together at **runtime**

---

## Integration Strategies

There are three ways to compose microfrontends. They differ in _when_ the composition happens.

### 1. Build-Time Integration

MFEs are published as npm packages. The shell installs them as dependencies and bundles them together.

```
Team A publishes: @company/search-mfe@1.2.0
Team B publishes: @company/catalog-mfe@3.1.0

Shell:
  package.json dependencies:
    "@company/search-mfe": "1.2.0"
    "@company/catalog-mfe": "3.1.0"
```

```
Build time:
Shell + Search MFE + Catalog MFE ──► Single bundle ──► Deploy
```

**Problem:** This isn't really a microfrontend. If Team A ships a fix, the shell must update its dependency, rebuild, and redeploy. Teams are still coupled at release time.

✗ Not recommended for true team independence.

---

### 2. Runtime Integration via Module Federation

Teams deploy their own bundles independently. The shell loads them dynamically at runtime.

```
Team A deploys:  https://cdn.company.com/search/remoteEntry.js
Team B deploys:  https://cdn.company.com/catalog/remoteEntry.js

Shell loads them at runtime — no rebuild needed when teams ship
```

This is the dominant modern approach. Powered by **Webpack Module Federation** or **Vite Federation**.

---

### 3. Runtime Integration via iframes

Each MFE runs in its own iframe. The shell composes them on the page.

```html
<iframe src="https://search.company.com" />
<iframe src="https://catalog.company.com" />
```

Strongest isolation (each app is completely separate), but terrible UX:

- Shared state is nearly impossible
- Consistent styling is very hard
- Performance overhead from multiple full page loads

✗ Only appropriate for highly isolated third-party integrations (e.g. embedding a payment widget).

---

## Module Federation Deep Dive

Module Federation allows one JavaScript build to **expose** modules and another to **consume** them at runtime, without bundling them together.

```
┌─────────────────────────────────────────────────────────┐
│                   How It Works                          │
│                                                         │
│  Search MFE build output:                               │
│  ├── remoteEntry.js    ← manifest: "I expose SearchApp" │
│  ├── search.chunk.js                                    │
│  └── vendors.chunk.js                                   │
│                                                         │
│  Shell at runtime:                                      │
│  1. Loads remoteEntry.js from Search MFE's CDN URL      │
│  2. Reads manifest → "SearchApp is at search.chunk.js"  │
│  3. Dynamically loads search.chunk.js                   │
│  4. Renders <SearchApp /> as if it were local           │
└─────────────────────────────────────────────────────────┘
```

### Shell Configuration (Host)

```javascript
// shell/vite.config.ts
import federation from "@originjs/vite-plugin-federation";

export default {
  plugins: [
    federation({
      name: "shell",
      remotes: {
        // Where to find each MFE's remoteEntry at runtime
        searchMfe: "https://cdn.company.com/search/remoteEntry.js",
        catalogMfe: "https://cdn.company.com/catalog/remoteEntry.js",
        cartMfe: "https://cdn.company.com/cart/remoteEntry.js",
      },
      shared: ["react", "react-dom"], // loaded once, shared across all MFEs
    }),
  ],
};
```

### MFE Configuration (Remote)

```javascript
// search-mfe/vite.config.ts
import federation from "@originjs/vite-plugin-federation";

export default {
  plugins: [
    federation({
      name: "searchMfe",
      filename: "remoteEntry.js",
      exposes: {
        // What this MFE makes available to the shell
        "./SearchApp": "./src/SearchApp.tsx",
        "./SearchWidget": "./src/components/SearchWidget.tsx",
      },
      shared: ["react", "react-dom"],
    }),
  ],
};
```

### Consuming in the Shell

```tsx
// shell/src/pages/SearchPage.tsx
import { lazy, Suspense } from "react";

// This import resolves at runtime from the remote CDN
const SearchApp = lazy(() => import("searchMfe/SearchApp"));

export function SearchPage() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <SearchApp />
    </Suspense>
  );
}
```

The shell has no compile-time dependency on the Search MFE. Team A can deploy a new version of `SearchApp` and the shell picks it up on the next page load — no shell rebuild or redeploy needed.

---

## Shared Dependencies Problem

The biggest technical challenge in microfrontends: if each MFE bundles its own copy of React, users download React multiple times.

```
Without sharing:
Shell loads React 18    (45kb)
Search MFE loads React 18 (45kb) ← duplicate
Catalog MFE loads React 18 (45kb) ← duplicate
Total: 135kb just for React

With Module Federation shared config:
React 18 loaded once (45kb) ← shared singleton
All MFEs use the same instance
Total: 45kb
```

```javascript
// Both shell and MFEs must declare the same shared config
shared: {
  react: {
    singleton: true,      // only one instance across all MFEs
    requiredVersion: '^18.0.0',
  },
  'react-dom': {
    singleton: true,
    requiredVersion: '^18.0.0',
  },
},
```

**Version mismatch problem:** If Shell requires React `^18.0.0` and Cart MFE ships React `^17.0.0`, Module Federation can't share them — it loads both. This is why **a shared platform team** managing dependency versions is critical at MFE scale.

---

## Cross-MFE Communication

MFEs are isolated. They don't share React context or Redux stores. Communication options:

### 1. Custom Events (simplest)

```typescript
// Cart MFE — fires an event when item is added
window.dispatchEvent(
  new CustomEvent("cart:item-added", {
    detail: { productId: "p1", quantity: 1 },
  }),
);

// Shell or another MFE — listens
window.addEventListener("cart:item-added", (event) => {
  updateCartBadge(event.detail.quantity);
});
```

Simple, framework-agnostic. The downside: event names are untyped strings — easy to break silently.

### 2. Shared State via a Singleton Store

```typescript
// shared-state package — published by platform team
// shell and all MFEs install this package

import { createStore } from '@company/shared-state';

const store = createStore({
  cart: { items: [], count: 0 },
  auth: { user: null, token: null },
});

// Cart MFE
store.setState('cart', { items: [...], count: 3 });

// Shell
store.subscribe('cart', (cart) => updateBadge(cart.count));
```

More structured than events, but requires a shared package all MFEs depend on — introducing coupling through the back door.

### 3. URL as Shared State

The most decoupled option. MFEs communicate via URL params and query strings — no shared runtime dependency.

```
/checkout?cartId=abc123&userId=u456

Checkout MFE reads cartId from URL
Auth info comes from the URL or a cookie
No direct MFE-to-MFE communication needed
```

---

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Deployment Flow                              │
│                                                                 │
│  Team A (Search):                                               │
│  Code → CI → Build → Deploy to CDN ──► search/remoteEntry.js    │
│                                        (independent, any time)  │
│                                                                 │
│  Team B (Catalog):                                              │
│  Code → CI → Build → Deploy to CDN ──► catalog/remoteEntry.js   │
│                                        (independent, any time)  │
│                                                                 │
│  Platform Team (Shell):                                         │
│  Code → CI → Build → Deploy to CDN ──► shell/index.html         │
│                       (only changes when shell itself changes)  │
│                                                                 │
│  User loads shell → shell loads latest MFEs from CDN            │
└─────────────────────────────────────────────────────────────────┘
```

Teams ship independently. A bug fix in Search goes live in minutes — no coordination with Catalog or Cart teams.

---

## Trade-offs

|                           | Feature-based Monolith   | Microfrontend                         |
| ------------------------- | ------------------------ | ------------------------------------- |
| **Team independence**     | Medium                   | High                                  |
| **Independent deploy**    | ✗                        | ✓                                     |
| **Setup complexity**      | Low                      | High                                  |
| **Shared state**          | Easy (one store)         | Hard (custom events, singletons)      |
| **Consistent UI**         | Easy (one design system) | Requires shared package               |
| **Performance**           | One bundle, optimized    | Multiple bundles, shared dep overhead |
| **DX / debugging**        | Straightforward          | Complex (cross-repo, runtime errors)  |
| **Framework flexibility** | One framework            | Multiple possible                     |
| **Recommended team size** | 2–20 engineers           | 20+ engineers, multiple product teams |
