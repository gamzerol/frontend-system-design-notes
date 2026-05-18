# Monolith vs Modular Frontend Architecture

A monolithic frontend bundles all features into a single, tightly coupled application. A modular frontend splits the application into independent, loosely coupled units. Neither is inherently better — the right choice depends on team size, product complexity, and how fast things need to move.

---

## The Monolith

A monolithic frontend is a single deployable unit where all features share the same codebase, build pipeline, and runtime.

```
┌──────────────────────────────────────────────┐
│              Single SPA Bundle               │
│                                              │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│   │  Auth    │  │  Cart    │  │ Checkout │   │
│   └──────────┘  └──────────┘  └──────────┘   │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│   │ Products │  │ Profile  │  │ Search   │   │
│   └──────────┘  └──────────┘  └──────────┘   │
│                                              │
│          Shared: State, Router, Utils        │
└──────────────────────────────────────────────┘
                      │
                 Single deploy
                      │
                   CDN / Server
```

### When monolith works well

- Small-to-medium product with a single team (< 5–8 engineers)
- Early stage: requirements change fast, iteration speed matters most
- Shared UI consistency is critical (one design system, one router)
- You don't yet know where the natural feature boundaries are

### When monolith becomes painful

- A bug in the `Auth` module causes a full redeploy that affects `Checkout`
- 10 engineers all touching `src/components` causes constant merge conflicts
- One slow-loading feature slows down the entire initial bundle
- You want to A/B test a complete feature rewrite without touching the rest of the app

---

## The Modular Frontend

A modular frontend splits the application into independently developed (and sometimes independently deployed) units called **modules** or **features**.

```
┌─────────────────────────────────────────────────────┐
│                    Shell / Host App                 │
│              (routing, layout, auth guard)          │
└──────────┬──────────────┬──────────────┬────────────┘
           │              │              │
     ┌─────▼──────┐ ┌─────▼──────┐ ┌────▼───────┐
     │  Products  │ │   Cart &   │ │  Account   │
     │   Module   │ │  Checkout  │ │   Module   │
     │            │ │   Module   │ │            │
     │ Own state  │ │ Own state  │ │ Own state  │
     │ Own routes │ │ Own routes │ │ Own routes │
     │ Own tests  │ │ Own tests  │ │ Own tests  │
     └────────────┘ └────────────┘ └────────────┘
           │              │              │
     ┌─────▼──────────────▼──────────────▼────────┐
     │            Shared Package Layer            │
     │    (design system, API client, auth utils) │
     └────────────────────────────────────────────┘
```

Each module:

- Can be developed by a separate team
- Has its own internal state management
- Can be deployed independently (in a microfrontend setup)
- Fails independently — a broken module doesn't crash the shell

---

## Three Points on the Spectrum

Modular architecture isn't a binary switch. It's a spectrum:

```
Monolith ─────────────────────────────────────────► Microfrontend
    │                    │                                  │
 All in one          Feature-based                  Separate deployments
 shared state         folders, lazy                 Module Federation
 one bundle           loaded chunks                 separate teams
                    (most common)
```

**1. Classic Monolith**
Everything in `src/`, shared global state, one bundle. Fine for early-stage apps.

**2. Feature-based Monolith** ← _the sweet spot for most teams_
Same repo and deployment, but code is organized by feature with strict internal boundaries. Lazy loading splits the bundle per feature.

**3. Microfrontend**
Separate repos, separate build pipelines, runtime composition via Module Federation or iframes. Only justified at large team scale (20+ engineers, multiple product teams).

> Most applications should live at point **2**. The jump to microfrontends has real costs — don't make it prematurely.

---

## Feature-Based Folder Structure

The most impactful change you can make in a growing codebase is moving from **type-based** to **feature-based** organization.

**Type-based (doesn't scale):**

```
src/
├── components/
│   ├── Button.tsx
│   ├── ProductCard.tsx
│   └── CartItem.tsx
├── hooks/
│   ├── useProducts.ts
│   └── useCart.ts
├── store/
│   ├── productsSlice.ts
│   └── cartSlice.ts
└── pages/
    ├── ProductsPage.tsx
    └── CartPage.tsx
```

When you work on the Cart feature you touch 4 different folders. Context switching is constant.

**Feature-based (scales well):**

```
src/
├── features/
│   ├── products/
│   │   ├── components/
│   │   │   └── ProductCard.tsx
│   │   ├── hooks/
│   │   │   └── useProducts.ts
│   │   ├── store/
│   │   │   └── productsSlice.ts
│   │   ├── api/
│   │   │   └── productsApi.ts
│   │   └── index.ts          ← public API of this feature
│   │
│   └── cart/
│       ├── components/
│       │   └── CartItem.tsx
│       ├── hooks/
│       │   └── useCart.ts
│       ├── store/
│       │   └── cartSlice.ts
│       └── index.ts
│
├── shared/                   ← only truly shared things
│   ├── ui/                   ← design system components
│   ├── lib/                  ← utilities, API client
│   └── types/                ← shared TypeScript types
│
└── app/
    ├── router.tsx
    ├── store.ts
    └── App.tsx
```

The `index.ts` at the root of each feature is critical — it acts as the **public API boundary**.

---

## The Index Barrel: Enforcing Module Boundaries

Each feature exposes only what other features need. Everything else is private.

```typescript
// features/cart/index.ts — public API of the cart feature
export { CartButton } from "./components/CartButton";
export { CartDrawer } from "./components/CartDrawer";
export { useCartCount } from "./hooks/useCartCount";
export type { CartItem } from "./types";

// CartItem.tsx internals, cartSlice.ts, etc. are NOT exported
// Other features cannot import them directly
```

```typescript
// ✅ Correct — importing through the public API
import { CartButton } from "@/features/cart";

// ❌ Wrong — reaching into internals
import { CartItem } from "@/features/cart/components/CartItem";
```

This boundary prevents the accidental coupling that makes monoliths painful. You can refactor `CartItem.tsx` without worrying that something else in the codebase imports it directly.

> **Enforce this with ESLint:** The `eslint-plugin-boundaries` plugin can statically prevent cross-feature internal imports.

---

## Communication Between Modules

When modules need to talk to each other, there are three patterns:

**1. Shared state (simplest, most coupling)**
Both modules read/write to the same global store slice.

```typescript
// products feature reads cart state to show "added" indicator
import { useCartStore } from "@/features/cart";
```

Fine for tightly related features. Avoid for unrelated ones.

**2. Callback props / event bus (decoupled)**
The shell passes callbacks down. Modules emit events, don't import each other.

```typescript
// Shell wires modules together
<ProductCard
  onAddToCart={(product) => cartStore.add(product)}
/>
```

**3. URL as shared state (most decoupled)**
Features communicate through URL params and query strings.

```
/checkout?cartId=abc123&promoCode=SAVE10
```

Each module reads what it needs from the URL. No direct coupling.

---

## Lazy Loading: Splitting the Bundle by Feature

A modular structure unlocks route-based code splitting — each feature loads only when the user navigates to it.

```tsx
// app/router.tsx
import { lazy, Suspense } from "react";

const ProductsPage = lazy(() => import("@/features/products/ProductsPage"));
const CartPage = lazy(() => import("@/features/cart/CartPage"));
const CheckoutPage = lazy(() => import("@/features/checkout/CheckoutPage"));

export function AppRouter() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/products" element={<ProductsPage />} />
        <Route path="/cart" element={<CartPage />} />
        <Route path="/checkout" element={<CheckoutPage />} />
      </Routes>
    </Suspense>
  );
}
```

Result: a user visiting `/products` never downloads the checkout bundle. Initial load is faster for everyone.

---

## Trade-offs

|                        | Monolith      | Feature-based Monolith | Microfrontend                         |
| ---------------------- | ------------- | ---------------------- | ------------------------------------- |
| **Setup cost**         | Low           | Low–Medium             | High                                  |
| **Iteration speed**    | Fast          | Fast                   | Slow (cross-repo coordination)        |
| **Bundle splitting**   | Manual        | Route-based            | Per-app                               |
| **Independent deploy** | ✗             | ✗                      | ✓                                     |
| **Team autonomy**      | Low           | Medium                 | High                                  |
| **Shared state**       | Easy          | Easy                   | Hard                                  |
| **Consistent UI**      | Easy          | Easy                   | Requires shared design system package |
| **Recommended for**    | < 5 engineers | 5–20 engineers         | 20+ engineers, multiple teams         |

---

## Migration Path: Monolith → Modular

You don't rewrite — you migrate incrementally:

```
Step 1: Create /features directory
        Move one self-contained feature (e.g. auth) into it.
        Add index.ts barrel.

Step 2: Add ESLint boundaries rule.
        Fix all violations — this reveals hidden coupling.

Step 3: Add lazy() to the migrated feature's route.
        Measure bundle size improvement.

Step 4: Repeat for next feature.
        The codebase improves with each iteration.
```

> **Rule of thumb:** Never do a big-bang rewrite. Migrate one feature per sprint until the structure is in place.
