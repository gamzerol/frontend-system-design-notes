# Feature-Based Architecture

Feature-based architecture organizes code by **what it does** (business domain) rather than **what it is** (technical type). Instead of `components/`, `hooks/`, `utils/` folders at the root, you have `features/auth/`, `features/cart/`, `features/checkout/`. Each feature is a self-contained vertical slice — it owns its components, hooks, API calls, state, and tests.

---

## The Core Problem: Type-Based Organization Doesn't Scale

Most React projects start with type-based organization:

```
src/
├── components/       ← all components from all features
├── hooks/            ← all hooks from all features
├── store/            ← all store slices
├── utils/            ← all utility functions
├── services/         ← all API calls
└── pages/            ← all page components
```

This feels clean at first. At 5 features and 3 engineers, it works fine.

At 15 features and 10 engineers, it becomes painful:

```
Working on the Cart feature requires touching:
  components/CartItem.tsx
  components/CartSummary.tsx
  hooks/useCart.ts
  hooks/useCartTotal.ts
  store/cartSlice.ts
  services/cartService.ts
  utils/formatPrice.ts
  pages/CartPage.tsx

8 different folders for one feature.
Every PR touches shared folders → merge conflicts.
No clear ownership → "who broke the cart?"
```

---

## Feature-Based Organization

Group everything by feature domain:

```
src/
├── features/
│   ├── auth/
│   │   ├── components/
│   │   │   ├── LoginForm.tsx
│   │   │   └── AuthGuard.tsx
│   │   ├── hooks/
│   │   │   └── useAuth.ts
│   │   ├── api/
│   │   │   └── authApi.ts
│   │   ├── store/
│   │   │   └── authSlice.ts
│   │   ├── types/
│   │   │   └── index.ts
│   │   └── index.ts              ← public API
│   │
│   ├── cart/
│   │   ├── components/
│   │   │   ├── CartItem.tsx
│   │   │   ├── CartSummary.tsx
│   │   │   └── CartDrawer.tsx
│   │   ├── hooks/
│   │   │   ├── useCart.ts
│   │   │   └── useCartTotal.ts
│   │   ├── api/
│   │   │   └── cartApi.ts
│   │   ├── store/
│   │   │   └── cartSlice.ts
│   │   ├── lib/
│   │   │   └── formatPrice.ts
│   │   ├── types/
│   │   │   └── index.ts
│   │   └── index.ts
│   │
│   └── checkout/
│       ├── components/
│       ├── hooks/
│       ├── api/
│       └── index.ts
│
├── shared/                       ← only truly shared code
│   ├── ui/                       ← design system primitives
│   │   ├── Button/
│   │   ├── Input/
│   │   └── Modal/
│   ├── lib/                      ← shared utilities
│   │   ├── httpClient.ts
│   │   └── dateUtils.ts
│   └── types/                    ← shared TypeScript types
│
└── app/                          ← app-level wiring
    ├── router.tsx
    ├── store.ts
    └── App.tsx
```

Working on the Cart feature now means touching only `features/cart/`. One folder, clear ownership, minimal conflicts.

---

## The Vertical Slice

Feature-based architecture is an implementation of **vertical slice** thinking. Instead of horizontal layers that cut across features, each vertical slice owns the full stack from UI to API call.

```
Horizontal (type-based):          Vertical (feature-based):

components/ ─────────────────     ┌──────┐ ┌──────┐ ┌──────────┐
hooks/       ─────────────────     │ Auth │ │ Cart │ │ Checkout │
store/       ─────────────────     │      │ │      │ │          │
api/         ─────────────────     │  UI  │ │  UI  │ │    UI    │
                                  │ Hook │ │ Hook │ │   Hook   │
Touching one feature =            │ Store│ │ Store│ │  Store   │
touching all horizontal layers    │  API │ │  API │ │   API    │
                                  └──────┘ └──────┘ └──────────┘

                                  Touching one feature =
                                  touching one vertical column
```

---

## The Public API: index.ts

The most important file in each feature is its `index.ts`. It defines what the feature **exposes** to the rest of the app. Everything not exported is private.

```typescript
// features/cart/index.ts

// Components other features/pages can use
export { CartDrawer } from "./components/CartDrawer";
export { CartButton } from "./components/CartButton";

// Hooks other features/pages can use
export { useCartCount } from "./hooks/useCart";

// Types other features need
export type { CartItem, CartSummary } from "./types";

// CartItem.tsx, cartSlice.ts internals — NOT exported
// Other features cannot reach into cart internals
```

```typescript
// ✅ Correct — import through public API
import { CartButton, useCartCount } from "@/features/cart";

// ❌ Violation — reaching into internals
import { CartItem } from "@/features/cart/components/CartItem";
import { cartSlice } from "@/features/cart/store/cartSlice";
```

This boundary is what gives feature-based architecture its power. A team can refactor the entire internals of `features/cart/` without breaking anything else — as long as the `index.ts` contract stays the same.

---

## Enforcing Boundaries with ESLint

Boundaries only work if they're enforced. Document conventions are ignored under deadline pressure. ESLint rules are not.

```bash
npm install --save-dev eslint-plugin-boundaries
```

```javascript
// eslint.config.js
import boundaries from "eslint-plugin-boundaries";

export default [
  {
    plugins: { boundaries },
    settings: {
      "boundaries/elements": [
        { type: "app", pattern: "src/app/*" },
        { type: "feature", pattern: "src/features/*" },
        { type: "shared", pattern: "src/shared/*" },
      ],
    },
    rules: {
      // features cannot import from app/
      "boundaries/element-types": [
        "error",
        {
          default: "disallow",
          rules: [
            { from: "app", allow: ["feature", "shared"] },
            { from: "feature", allow: ["shared"] }, // features can't import each other's internals
            { from: "shared", allow: [] },
          ],
        },
      ],
    },
  },
];
```

Now cross-feature internal imports fail at lint time, not at code review.

---

## Cross-Feature Communication

Features are isolated, but they still need to interact. Three patterns, in order of increasing decoupling:

### 1. Shared State (simplest)

Both features read/write from a shared store slice. Appropriate for tightly related features.

```typescript
// features/products/components/ProductCard.tsx
import { useCartCount } from '@/features/cart'; // via public API only

function ProductCard({ product }) {
  const cartCount = useCartCount();
  return <div>{product.name} ({cartCount} in cart)</div>;
}
```

### 2. Callback Props via the Shell

The page/shell wires features together. Features don't know about each other.

```tsx
// app/pages/ProductsPage.tsx — shell wires features together
import { ProductList } from "@/features/products";
import { useCart } from "@/features/cart";

export function ProductsPage() {
  const { addItem } = useCart();

  return (
    <ProductList
      onAddToCart={(product) => addItem(product)} // cart logic stays in cart
    />
  );
}
```

`ProductList` doesn't import anything from `features/cart`. The shell is the only thing that knows about both.

### 3. URL / Query Params (most decoupled)

Features communicate through the URL. Each feature reads its own slice of URL state.

```
/checkout?promoCode=SAVE20&returnUrl=/products

// Checkout feature reads promoCode from URL
// Products feature reads returnUrl from URL
// Neither imports the other
```

---

## What Goes in `shared/`?

`shared/` is for code used by **three or more features**. It's not a dumping ground.

```
shared/ui/          ← Design system: Button, Input, Modal, Spinner
                      These have no business logic, only visual behavior

shared/lib/         ← Infrastructure: httpClient, dateUtils, validators
                      No feature-specific logic

shared/types/       ← Types shared across features: User, Pagination, ApiError
```

**Red flags for `shared/`:**

- A component that knows about `Order` or `Cart` — that's feature-specific
- A hook that calls a specific API endpoint — that belongs in a feature
- More than ~20 files — it's becoming a second `src/` root

> **The rule of three:** code should be duplicated in two features before being moved to `shared/`. Premature abstraction creates coupling. Duplication is cheaper than the wrong abstraction.

---

## Feature Flags Integration

Feature-based architecture maps naturally to feature flags — you can gate an entire feature behind a flag:

```tsx
// app/router.tsx
import { useFeatureFlag } from "@/shared/lib/featureFlags";

function AppRouter() {
  const newCheckout = useFeatureFlag("new-checkout-flow");

  return (
    <Routes>
      <Route
        path="/checkout"
        element={
          newCheckout ? (
            <NewCheckoutPage /> // features/checkout-v2
          ) : (
            <CheckoutPage />
          )
        } // features/checkout
      />
    </Routes>
  );
}
```

Each version is a separate, self-contained feature folder. No `if (newFeature)` scattered across 20 files.

---

## Compared to Other Approaches

|                          | Type-based        | Feature-based      | Feature-Sliced Design            |
| ------------------------ | ----------------- | ------------------ | -------------------------------- |
| **Organization**         | By technical type | By business domain | By domain + strict layer rules   |
| **Colocation**           | ✗ — scattered     | ✓ — together       | ✓ — together + formalized        |
| **Boundary enforcement** | None              | ESLint plugin      | Built-in methodology             |
| **Learning curve**       | Low               | Low–Medium         | Medium–High                      |
| **Best for**             | Small apps        | Most apps          | Large teams wanting strict rules |
