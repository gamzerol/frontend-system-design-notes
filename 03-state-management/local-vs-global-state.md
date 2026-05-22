# Local vs Global State

Not all state is equal. Before reaching for a global store, ask: "Who needs this state?" If the answer is one component or a small subtree, keep it local. Global state is for data that genuinely needs to be shared across distant parts of the application. Most state in a well-designed app should be local — or not in the client at all (server state).

---

## The Four Types of State

Before deciding _where_ state lives, identify _what kind_ of state it is:

```
┌─────────────────────────────────────────────────────────────────┐
│                      Types of State                             │
│                                                                 │
│  1. Local UI State       2. Shared UI State                     │
│     isOpen, inputValue      selectedTab, activeFilters          │
│     Lives in component      Lives in lifted state / context     │
│                                                                 │
│  3. Global App State     4. Server State                        │
│     auth user, theme        Products, orders, user profile      │
│     notifications           Lives on the server, cached locally │
│     Lives in global store   Managed by React Query / SWR        │
└─────────────────────────────────────────────────────────────────┘
```

The most common mistake: treating **server state** as global app state and putting API responses into Redux/Zustand. Server state has different needs — it goes stale, needs background refetching, and is owned by the backend. It deserves its own tool (React Query, SWR).

---

## The State Decision Tree

When you need to manage state, walk this tree:

```
Is this data from the server / an API?
│
├── YES → Use React Query / SWR (server state)
│         Don't put it in Redux or Zustand
│
└── NO → Is it UI state?
         │
         ├── Does only ONE component need it?
         │   └── useState / useReducer (local state)
         │
         ├── Do a FEW nearby components need it?
         │   └── Lift state up to common ancestor
         │       or use React Context (shared local state)
         │
         └── Do DISTANT or MANY components need it?
             │
             ├── Is it truly global? (auth, theme, notifications)
             │   └── Global store (Zustand, Redux)
             │
             └── Can it live in the URL?
                 └── URL state (query params, route params)
                     Shareable, bookmarkable, free persistence
```

---

## Local State: useState and useReducer

The default choice. Start here, move up only when you have a reason.

```tsx
// Simple local state — form input, toggle, counter
function SearchBar() {
  const [query, setQuery] = useState("");
  const [isFocused, setIsFocused] = useState(false);

  return (
    <input
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      onFocus={() => setIsFocused(true)}
      onBlur={() => setIsFocused(false)}
      className={isFocused ? "focused" : ""}
    />
  );
}
```

When local state gets complex (multiple sub-values, transitions between states), `useReducer` is cleaner:

```tsx
// useReducer for complex local state
type State = {
  step: "idle" | "loading" | "success" | "error";
  data: Order | null;
  error: string | null;
};

type Action =
  | { type: "SUBMIT" }
  | { type: "SUCCESS"; payload: Order }
  | { type: "ERROR"; payload: string };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "SUBMIT":
      return { ...state, step: "loading", error: null };
    case "SUCCESS":
      return { step: "success", data: action.payload, error: null };
    case "ERROR":
      return { ...state, step: "error", error: action.payload };
    default:
      return state;
  }
}

function CheckoutForm() {
  const [state, dispatch] = useReducer(reducer, {
    step: "idle",
    data: null,
    error: null,
  });

  async function handleSubmit() {
    dispatch({ type: "SUBMIT" });
    try {
      const order = await submitOrder();
      dispatch({ type: "SUCCESS", payload: order });
    } catch (err) {
      dispatch({ type: "ERROR", payload: err.message });
    }
  }

  // ...
}
```

---

## Lifting State Up

When two sibling components need the same state, lift it to their closest common ancestor.

```
Before (each component manages its own — can get out of sync):

ProductList ──── selectedId (local)
ProductDetail ── selectedId (local, duplicate)


After (lifted to parent — single source of truth):

ProductsPage
    ├── selectedId (lifted here)
    ├── ProductList ──── receives selectedId, onSelect
    └── ProductDetail ── receives selectedId
```

```tsx
// Lifted state pattern
function ProductsPage() {
  const [selectedId, setSelectedId] = useState<string | null>(null);

  return (
    <div className="layout">
      <ProductList selectedId={selectedId} onSelect={setSelectedId} />
      <ProductDetail productId={selectedId} />
    </div>
  );
}
```

**Lift only as high as needed.** If you lift to the root component when only two siblings need it, you cause unnecessary re-renders across the tree.

---

## React Context: Shared State Without Prop Drilling

Context solves prop drilling — passing props through many layers that don't use them.

```
Without context (prop drilling):
App
└── Layout (passes theme down — doesn't use it)
    └── Sidebar (passes theme down — doesn't use it)
        └── NavItem (actually uses theme)


With context:
App (provides theme)
└── Layout
    └── Sidebar
        └── NavItem (consumes theme directly)
```

```tsx
// Context definition
type ThemeContextValue = {
  theme: "light" | "dark";
  toggle: () => void;
};

const ThemeContext = createContext<ThemeContextValue | null>(null);

// Provider — wrap at the appropriate level (not always root)
export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<"light" | "dark">("light");

  return (
    <ThemeContext.Provider
      value={{
        theme,
        toggle: () => setTheme((t) => (t === "light" ? "dark" : "light")),
      }}
    >
      {children}
    </ThemeContext.Provider>
  );
}

// Custom hook — always wrap context in a hook
export function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) throw new Error("useTheme must be used within ThemeProvider");
  return context;
}

// Consumer — anywhere in the tree
function NavItem() {
  const { theme, toggle } = useTheme();
  return <button onClick={toggle}>{theme === "light" ? "🌙" : "☀️"}</button>;
}
```

### Context Performance Gotcha

Every component consuming a context re-renders when **any** value in the context changes. Split contexts by update frequency:

```tsx
// ❌ One big context — theme change re-renders everything that uses auth too
const AppContext = createContext({ user, theme, notifications });

// ✅ Split by update frequency
const AuthContext = createContext({ user }); // changes rarely
const ThemeContext = createContext({ theme, toggle }); // changes on user action
const NotifContext = createContext({ notifications }); // changes frequently
```

---

## Global State: When Nothing Else Fits

Reserve global state for data that is:

- Needed by many unrelated components across the tree
- Truly app-wide (auth user, theme, feature flags, notifications)
- Not server data (use React Query for that)

```tsx
// Zustand — minimal global store
import { create } from "zustand";

type AuthState = {
  user: User | null;
  token: string | null;
  login: (user: User, token: string) => void;
  logout: () => void;
};

export const useAuthStore = create<AuthState>((set) => ({
  user: null,
  token: null,
  login: (user, token) => set({ user, token }),
  logout: () => set({ user: null, token: null }),
}));

// Usage — any component, anywhere in the tree
function Header() {
  const { user, logout } = useAuthStore();
  return user ? <button onClick={logout}>{user.name}</button> : <LoginLink />;
}
```

---

## URL State: The Underused Option

URL state is powerful and underused. It's free persistence, shareability, and browser history integration.

```tsx
// ❌ Storing filters in component state — lost on refresh, not shareable
const [category, setCategory] = useState('shoes');
const [sort, setSort] = useState('price-asc');
const [page, setPage] = useState(1);

// ✅ Storing filters in URL — shareable, bookmarkable, back-button works
// URL: /products?category=shoes&sort=price-asc&page=1

import { useSearchParams } from 'react-router-dom';

function ProductFilters() {
  const [params, setParams] = useSearchParams();

  const category = params.get('category') ?? 'all';
  const sort = params.get('sort') ?? 'relevance';

  function setCategory(value: string) {
    setParams(prev => { prev.set('category', value); return prev; });
  }

  return (/* filter UI */);
}
```

Good candidates for URL state: search queries, filters, sort order, pagination, selected tab, modal open/closed (if shareable).

---

## The State Ownership Pyramid

```
                    ┌───────────┐
                    │  Server   │   ← React Query / SWR
                    │  State    │     Most of your "data"
                    └─────┬─────┘
                          │ less data as you go down
                    ┌─────▼─────┐
                    │  Global   │   ← Zustand / Redux
                    │  UI State │     auth, theme, notifications
                    └─────┬─────┘
                          │
                    ┌─────▼─────┐
                    │  Shared   │   ← Context / lifted state
                    │  UI State │     filters, selected items
                    └─────┬─────┘
                          │
                    ┌─────▼─────┐
                    │  Local    │   ← useState / useReducer
                    │  UI State │     inputs, toggles, focus
                    └───────────┘

The pyramid should be wide at the bottom.
Most state should be local. Global state should be minimal.
```

---

## Common Mistakes

**1. Over-globalizing state**
Putting form input values, modal open/close, or hover states into Redux. These are local — they belong in `useState`.

**2. Using global store for server data**
Fetching products and putting them in Redux. React Query handles caching, background refetching, and stale data automatically — Redux doesn't.

**3. Prop drilling as a reason to go global**
Prop drilling through 3–4 levels is annoying but not a problem that needs a global store. Context solves this with less overhead.

**4. One giant context**
A single `AppContext` with everything in it. Every consumer re-renders on every change. Split by domain and update frequency.

**5. Ignoring the URL**
Storing filters and search queries in component state. A user can't share or bookmark their view. URL state is free.

---

## Quick Reference

| State type       | Tool              | Example                         |
| ---------------- | ----------------- | ------------------------------- |
| Local UI         | `useState`        | `isOpen`, `inputValue`, `count` |
| Complex local    | `useReducer`      | Multi-step form, loading states |
| Shared (nearby)  | Lifted state      | `selectedId` shared by siblings |
| Shared (distant) | React Context     | `theme`, `locale`               |
| Global app       | Zustand / Redux   | `auth.user`, `notifications`    |
| Server data      | React Query / SWR | Products, orders, user profile  |
| Shareable UI     | URL params        | Filters, search, pagination     |
