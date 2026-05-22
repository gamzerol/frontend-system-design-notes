# Redux vs Zustand (vs Jotai)

Redux is a predictable, opinionated state container with excellent DevTools and a large ecosystem — but comes with significant boilerplate. Zustand is a minimal, unopinionated alternative that solves the same problems with a fraction of the API surface. Jotai takes an atomic approach, closer to React's own model. For most applications, Zustand is the pragmatic choice. Redux shines at large team scale where conventions and strict structure matter more than flexibility.

---

## Why Global State Libraries Exist

React's built-in tools (useState, Context) have limitations at scale:

```
Problem 1: Context performance
  One AppContext with many values → every consumer re-renders on any change

Problem 2: Context update granularity
  Can't subscribe to a slice of context — it's all or nothing

Problem 3: State outside React
  Sometimes state needs to be read/written from outside the component tree
  (websocket handlers, router callbacks, utility functions)

Problem 4: DevTools & time-travel debugging
  useState gives no visibility into state changes over time
```

Global state libraries solve all four. The difference is _how_ they solve them.

---

## Redux (with Redux Toolkit)

Redux enforces a strict unidirectional data flow. State lives in a single store, changes happen only through dispatched actions, and reducers produce the next state immutably.

```
┌───────────────────────────────────────────────────────┐
│                    Redux Data Flow                    │
│                                                       │
│   Component                                           │
│      │                                                │
│      │ dispatch(action)                               │
│      ▼                                                │
│   Store ──► Reducer ──► New State ──► Component       │
│      │                                re-renders      │
│      │                                                │
│   Middleware (Redux Thunk / RTK Query)                │
│      │                                                │
│      └──► Side effects (API calls, logging)           │
└───────────────────────────────────────────────────────┘
```

### Redux Toolkit Setup

Modern Redux always uses Redux Toolkit (RTK). Vanilla Redux without RTK is not recommended.

```typescript
// features/cart/store/cartSlice.ts
import { createSlice, PayloadAction } from "@reduxjs/toolkit";

type CartItem = { id: string; name: string; price: number; quantity: number };
type CartState = { items: CartItem[]; isOpen: boolean };

const initialState: CartState = { items: [], isOpen: false };

const cartSlice = createSlice({
  name: "cart",
  initialState,
  reducers: {
    addItem(state, action: PayloadAction<CartItem>) {
      const existing = state.items.find((i) => i.id === action.payload.id);
      if (existing) {
        existing.quantity += 1; // Immer allows direct mutation
      } else {
        state.items.push(action.payload);
      }
    },
    removeItem(state, action: PayloadAction<string>) {
      state.items = state.items.filter((i) => i.id !== action.payload);
    },
    toggleCart(state) {
      state.isOpen = !state.isOpen;
    },
    clearCart(state) {
      state.items = [];
    },
  },
});

export const { addItem, removeItem, toggleCart, clearCart } = cartSlice.actions;
export default cartSlice.reducer;
```

```typescript
// app/store.ts
import { configureStore } from "@reduxjs/toolkit";
import cartReducer from "@/features/cart/store/cartSlice";
import authReducer from "@/features/auth/store/authSlice";

export const store = configureStore({
  reducer: {
    cart: cartReducer,
    auth: authReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

```typescript
// shared/lib/hooks.ts — typed hooks
import { useDispatch, useSelector } from "react-redux";
import type { RootState, AppDispatch } from "@/app/store";

export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector = <T>(selector: (state: RootState) => T) =>
  useSelector(selector);
```

```tsx
// Usage in component
function CartButton() {
  const dispatch = useAppDispatch();
  const count = useAppSelector((state) => state.cart.items.length);

  return <button onClick={() => dispatch(toggleCart())}>Cart ({count})</button>;
}
```

### Selectors with Reselect

Selectors memoize derived state — they only recompute when their inputs change:

```typescript
import { createSelector } from "@reduxjs/toolkit";

const selectCartItems = (state: RootState) => state.cart.items;

export const selectCartTotal = createSelector(selectCartItems, (items) =>
  items.reduce((sum, item) => sum + item.price * item.quantity, 0),
);

export const selectCartCount = createSelector(selectCartItems, (items) =>
  items.reduce((sum, item) => sum + item.quantity, 0),
);

// Component only re-renders when total actually changes
const total = useAppSelector(selectCartTotal);
```

---

## Zustand

Zustand takes the opposite philosophy: minimal API, no boilerplate, no conventions enforced. You define a store as a plain function and use it directly.

```typescript
// features/cart/store/cartStore.ts
import { create } from "zustand";
import { devtools, persist } from "zustand/middleware";

type CartItem = { id: string; name: string; price: number; quantity: number };

type CartStore = {
  items: CartItem[];
  isOpen: boolean;
  addItem: (item: CartItem) => void;
  removeItem: (id: string) => void;
  toggleCart: () => void;
  clearCart: () => void;
  total: () => number;
};

export const useCartStore = create<CartStore>()(
  devtools(
    persist(
      (set, get) => ({
        items: [],
        isOpen: false,

        addItem: (item) =>
          set((state) => {
            const existing = state.items.find((i) => i.id === item.id);
            if (existing) {
              return {
                items: state.items.map((i) =>
                  i.id === item.id ? { ...i, quantity: i.quantity + 1 } : i,
                ),
              };
            }
            return { items: [...state.items, item] };
          }),

        removeItem: (id) =>
          set((state) => ({ items: state.items.filter((i) => i.id !== id) })),

        toggleCart: () => set((state) => ({ isOpen: !state.isOpen })),

        clearCart: () => set({ items: [] }),

        // Derived state as a function — computed on access
        total: () =>
          get().items.reduce((sum, i) => sum + i.price * i.quantity, 0),
      }),
      { name: "cart-storage" }, // persists to localStorage
    ),
  ),
);
```

```tsx
// Usage — simpler than Redux, no Provider needed
function CartButton() {
  const { items, toggleCart } = useCartStore();

  return <button onClick={toggleCart}>Cart ({items.length})</button>;
}

// Granular subscriptions — component only re-renders when `items` changes
function CartTotal() {
  const total = useCartStore((state) => state.total());
  return <span>${total.toFixed(2)}</span>;
}
```

### Zustand Outside React

One of Zustand's strengths — the store is accessible outside the component tree:

```typescript
// In a WebSocket handler, router callback, or utility function
import { useCartStore } from "@/features/cart/store/cartStore";

const { clearCart } = useCartStore.getState();

// After logout
function handleLogout() {
  clearCart(); // clear cart
  router.push("/login");
}
```

---

## Jotai: Atomic State

Jotai models state as atoms — small, independent pieces that compose. Closer to React's own model.

```typescript
import { atom, useAtom } from 'jotai';

// Primitive atoms
const cartItemsAtom = atom<CartItem[]>([]);
const isCartOpenAtom = atom(false);

// Derived atom (computed, read-only)
const cartTotalAtom = atom(
  (get) => get(cartItemsAtom).reduce((sum, i) => sum + i.price * i.quantity, 0)
);

// Write atom (action)
const addItemAtom = atom(
  null,
  (get, set, item: CartItem) => {
    const items = get(cartItemsAtom);
    const existing = items.find(i => i.id === item.id);
    if (existing) {
      set(cartItemsAtom, items.map(i =>
        i.id === item.id ? { ...i, quantity: i.quantity + 1 } : i
      ));
    } else {
      set(cartItemsAtom, [...items, item]);
    }
  }
);

// Component subscribes only to atoms it uses
function CartTotal() {
  const [total] = useAtom(cartTotalAtom); // only re-renders when total changes
  return <span>${total.toFixed(2)}</span>;
}
```

Jotai shines when you have many small, independent pieces of global state that components subscribe to individually. It's the most granular re-render control of the three.

---

## Side-by-Side Comparison

Same feature implemented in each:

```
Feature: Add item to cart

Redux:
  1. Define CartItem type
  2. Create cartSlice with addItem reducer
  3. Export addItem action creator
  4. Add cartReducer to store
  5. Create typed useAppDispatch hook
  6. Component: dispatch(addItem(item))
  Files touched: cartSlice.ts, store.ts, hooks.ts, component

Zustand:
  1. Add addItem function to useCartStore
  2. Component: useCartStore(s => s.addItem)(item)
  Files touched: cartStore.ts, component

Jotai:
  1. Create addItemAtom
  2. Component: const [, addItem] = useAtom(addItemAtom)
  Files touched: cartAtoms.ts, component
```

---

## Comparison Table

|                        | Redux Toolkit              | Zustand                             | Jotai                         |
| ---------------------- | -------------------------- | ----------------------------------- | ----------------------------- |
| **Bundle size**        | ~16kb                      | ~1kb                                | ~3kb                          |
| **Boilerplate**        | Medium (RTK reduces it)    | Minimal                             | Minimal                       |
| **Learning curve**     | High                       | Low                                 | Medium                        |
| **DevTools**           | Excellent (Redux DevTools) | Good (via middleware)               | Good (Jotai DevTools)         |
| **Structure enforced** | Yes (actions, reducers)    | No                                  | No                            |
| **Re-render control**  | Selector-based             | Selector-based                      | Atom-based (most granular)    |
| **Outside React**      | Yes                        | Yes                                 | Limited                       |
| **Middleware**         | Rich ecosystem             | Built-in (devtools, persist, immer) | Plugin-based                  |
| **Team conventions**   | Enforced by structure      | Must define manually                | Must define manually          |
| **Best for**           | Large teams, complex flows | Most apps                           | Many independent state slices |

---

## When to Choose What

```
Choose Redux when:
  ✓ Large team (10+ engineers) — enforced conventions matter
  ✓ Complex async flows — RTK Query or Redux Saga
  ✓ Strict auditability needed — every state change logged as an action
  ✓ Already using it — migration cost isn't worth the switch

Choose Zustand when:
  ✓ Small-to-medium team (2–10 engineers)
  ✓ You want to move fast with minimal ceremony
  ✓ You need to access store outside React (event handlers, utils)
  ✓ Starting a new project — Zustand is the pragmatic default

Choose Jotai when:
  ✓ Many small, independent global state pieces
  ✓ Fine-grained re-render control is critical
  ✓ You like React's mental model and want atoms to feel natural
  ✓ Server Components / Next.js App Router (Jotai plays well here)
```
