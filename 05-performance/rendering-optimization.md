# Rendering Optimization

## TL;DR

Rendering optimization is about doing less work, less often, and at the right time. In React, this means understanding what triggers re-renders, how the reconciler works, and when memoization actually helps (hint: less often than you think). At the system level, it means choosing the right rendering strategy — CSR, SSR, SSG, or ISR — based on your data's update frequency and SEO requirements.

---

## How React Rendering Works

Before optimizing, understand the model:

```
┌─────────────────────────────────────────────────────────────────┐
│                    React Render Pipeline                        │
│                                                                 │
│  1. Trigger          2. Render Phase       3. Commit Phase      │
│  ──────────          ──────────────        ─────────────        │
│  State change        React calls your      React updates        │
│  Props change        component functions   the real DOM         │
│  Context change      Builds a new          Runs effects         │
│  forceUpdate         virtual DOM tree      (useEffect)          │
│                      Diffs with previous                        │
│                      (reconciliation)                           │
│                                                                 │
│  Render phase is pure and can be interrupted (Concurrent Mode)  │
│  Commit phase is synchronous and touches the real DOM           │
└─────────────────────────────────────────────────────────────────┘
```

Key insight: **rendering a component ≠ updating the DOM**. React renders (calls your function) frequently. It only commits (touches the DOM) when something actually changed. The DOM update is the expensive part — React's diffing minimizes it.

---

## What Triggers a Re-render

```
Component re-renders when:

  1. Its own state changes        useState, useReducer dispatch
  2. Its parent re-renders        even if props haven't changed
  3. A context it consumes changes
  4. A hook it uses changes       custom hook internal state

Component does NOT re-render when:

  ✗ Its sibling re-renders
  ✗ A context it doesn't consume changes
  ✗ A ref changes (useRef)
```

The most common source of unnecessary re-renders: **parent re-renders cascade to all children**, even if their props are identical.

```tsx
// Every time Counter updates, ALL children re-render
function Parent() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <button onClick={() => setCount((c) => c + 1)}>{count}</button>
      <ExpensiveList /> {/* re-renders on every count change */}
      <Sidebar /> {/* re-renders on every count change */}
      <Footer /> {/* re-renders on every count change */}
    </div>
  );
}
```

---

## Optimization 1: State Colocation

The best optimization is moving state down so fewer components are affected by its changes:

```tsx
// ❌ State at the top — every sibling re-renders on count change
function Parent() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <Counter count={count} onChange={setCount} />
      <ExpensiveList /> {/* unnecessary re-render */}
      <Sidebar /> {/* unnecessary re-render */}
    </div>
  );
}

// ✅ State colocated — only Counter re-renders
function Parent() {
  return (
    <div>
      <Counter /> {/* manages its own count */}
      <ExpensiveList /> {/* never re-renders */}
      <Sidebar /> {/* never re-renders */}
    </div>
  );
}

function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount((c) => c + 1)}>{count}</button>;
}
```

This is more effective than any memoization — you eliminate the re-render entirely rather than making it cheaper.

---

## Optimization 2: Children as Props

Pass expensive subtrees as `children` to prevent them from being re-created on parent re-render:

```tsx
// ❌ ExpensiveTree is a child of StatefulWrapper
// It re-renders every time StatefulWrapper's state changes
function App() {
  return (
    <StatefulWrapper>
      <ExpensiveTree /> {/* re-renders when wrapper re-renders */}
    </StatefulWrapper>
  );
}

// But wait — this IS the children as props pattern!
// React does NOT re-render children passed as props
// unless their own props change.

// The key: the element <ExpensiveTree /> is created in App,
// not in StatefulWrapper. App doesn't re-render just because
// StatefulWrapper re-renders.
```

```tsx
// Concrete example
function StatefulParent({ children }: { children: ReactNode }) {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <div>
      <button onClick={() => setIsOpen(o => !o)}>Toggle</button>
      {isOpen && <Drawer />}
      {children}   {/* children don't re-render when isOpen changes */}
    </div>
  );
}

function App() {
  return (
    <StatefulParent>
      <ExpensiveTree />   {/* created in App, owned by App */}
    </StatefulParent>     {/* StatefulParent re-rendering doesn't affect it */}
  );
}
```

---

## Optimization 3: React.memo

`React.memo` prevents a component from re-rendering when its props haven't changed (shallow comparison):

```tsx
// Without memo — re-renders whenever parent re-renders
function ProductCard({ product }: { product: Product }) {
  return (
    <div>
      {product.name} — ${product.price}
    </div>
  );
}

// With memo — only re-renders when product prop changes
const ProductCard = memo(function ProductCard({
  product,
}: {
  product: Product;
}) {
  return (
    <div>
      {product.name} — ${product.price}
    </div>
  );
});
```

### When memo actually helps

```
memo is useful when:
  ✓ Component renders often (parent re-renders frequently)
  ✓ Component is expensive to render (complex tree, heavy computation)
  ✓ Props are stable (primitives, or memoized objects/functions)

memo is wasted when:
  ✗ Props change every render anyway (memo check cost > render cost)
  ✗ Component is trivially cheap to render
  ✗ Props include new object/function references every render
```

The last point is critical. `memo` uses shallow comparison. If a prop is an object or function created inline, it's a new reference every render — `memo` won't help:

```tsx
// ❌ memo does nothing — new object reference every render
<ProductCard
  product={{ id: "p1", name: "Keyboard" }} // new object each render
  onClick={() => handleClick("p1")} // new function each render
/>;

// ✅ memo works — stable references
const product = useMemo(() => ({ id: "p1", name: "Keyboard" }), []);
const handleClick = useCallback(() => handleClick("p1"), []);
<ProductCard product={product} onClick={handleClick} />;
```

---

## Optimization 4: useMemo and useCallback

Both memoize across renders. Use them to stabilize references passed to memoized components or used as effect dependencies.

```tsx
// useMemo — memoize expensive computed values
function ProductList({ products, filters }: Props) {
  // Only recomputes when products or filters change
  const filtered = useMemo(
    () =>
      products
        .filter((p) => p.category === filters.category)
        .sort((a, b) => a.price - b.price),
    [products, filters.category],
  );

  return (
    <ul>
      {filtered.map((p) => (
        <ProductCard key={p.id} product={p} />
      ))}
    </ul>
  );
}

// useCallback — memoize functions passed as props to memo'd components
function Parent() {
  const [items, setItems] = useState<string[]>([]);

  // Without useCallback — new function reference every render
  // → MemoizedChild re-renders even though nothing changed
  const handleAdd = useCallback(
    (item: string) => setItems((prev) => [...prev, item]),
    [], // setItems is stable, no deps needed
  );

  return <MemoizedChild onAdd={handleAdd} />;
}
```

### The golden rule of memoization

```
Don't add useMemo/useCallback speculatively.
Profile first. Add memoization only where you've measured a problem.

Memoization has costs:
  - Extra memory (storing previous values)
  - Cognitive overhead (dependency arrays)
  - Can mask architecture problems that should be fixed instead
```

---

## Optimization 5: Key Prop and List Rendering

React uses `key` to match elements between renders. Wrong keys cause unnecessary unmount/remount:

```tsx
// ❌ Index as key — causes re-renders and state bugs when list order changes
{
  products.map((product, index) => (
    <ProductCard key={index} product={product} />
  ));
}

// ✅ Stable unique ID as key
{
  products.map((product) => <ProductCard key={product.id} product={product} />);
}
```

Using index as key only works for static lists that never reorder.

---

## Rendering Strategies

Beyond component-level optimization, the rendering strategy determines where and when HTML is generated.

```
┌────────────────┬──────────────────────────────────────────────────────┐
│   Strategy     │   How it works                                        │
├────────────────┼──────────────────────────────────────────────────────┤
│ CSR            │ Server sends empty HTML + JS bundle.                  │
│ Client-Side    │ Browser downloads JS, runs it, renders HTML.          │
│ Rendering      │ → Slow first load, fast subsequent navigation.        │
├────────────────┼──────────────────────────────────────────────────────┤
│ SSR            │ Server renders HTML per request. Browser receives     │
│ Server-Side    │ full HTML, hydrates with JS.                          │
│ Rendering      │ → Fast first load, server cost per request.           │
├────────────────┼──────────────────────────────────────────────────────┤
│ SSG            │ HTML generated at build time. Served as static files. │
│ Static Site    │ → Fastest possible, CDN-cacheable, can't be dynamic.  │
│ Generation     │                                                       │
├────────────────┼──────────────────────────────────────────────────────┤
│ ISR            │ SSG pages regenerated in the background on a timer    │
│ Incremental    │ or on-demand. Stale-while-revalidate for pages.       │
│ Static Regen.  │ → SSG speed + dynamic content. Next.js feature.       │
├────────────────┼──────────────────────────────────────────────────────┤
│ Streaming SSR  │ Server sends HTML in chunks as it renders.            │
│                │ Browser shows content before full page is ready.      │
│                │ → Best perceived performance for complex pages.       │
└────────────────┴──────────────────────────────────────────────────────┘
```

### Choosing a Rendering Strategy

```
Data changes constantly + SEO matters → SSR
  Examples: news article, product page, social feed

Data changes rarely + SEO matters → SSG or ISR
  Examples: blog post, marketing page, docs

Data is user-specific + no SEO needed → CSR
  Examples: dashboard, admin panel, SaaS app

Some pages static, some dynamic → mix strategies
  Next.js App Router lets you choose per route
```

---

## Core Web Vitals

Google's metrics for real-world rendering performance. They directly affect search ranking.

```
┌────────────────────────────────────────────────────────────────┐
│                     Core Web Vitals                            │
│                                                                │
│  LCP — Largest Contentful Paint                                │
│  How long until the main content is visible?                   │
│  Good: < 2.5s   |   Needs work: 2.5–4s   |   Poor: > 4s        │
│  Fix: SSR/SSG, image optimization, preload key resources       │
│                                                                │
│  INP — Interaction to Next Paint (replaced FID)                │
│  How fast does the page respond to user input?                 │
│  Good: < 200ms  |   Needs work: 200–500ms |   Poor: > 500ms    │
│  Fix: reduce JS execution, defer non-critical work             │
│                                                                │
│  CLS — Cumulative Layout Shift                                 │
│  How much does the page jump around while loading?             │
│  Good: < 0.1    |   Needs work: 0.1–0.25  |   Poor: > 0.25     │
│  Fix: reserve space for images/ads, avoid injecting content    │
└────────────────────────────────────────────────────────────────┘
```

```tsx
// Measuring Web Vitals in React
import { onLCP, onINP, onCLS } from "web-vitals";

onLCP((metric) => analytics.track("LCP", metric));
onINP((metric) => analytics.track("INP", metric));
onCLS((metric) => analytics.track("CLS", metric));
```

---

## Common Profiling Workflow

```
1. Measure first, optimize second
   Open React DevTools Profiler
   Record a user interaction (click, scroll, type)
   Identify components with long render times

2. Check what's causing the re-render
   "Why did this render?" in React DevTools
   Most common: parent re-rendered, context changed

3. Apply the right fix (in order of preference)
   a. Colocate state → eliminate the re-render
   b. Children as props → prevent cascade
   c. React.memo → skip re-render when props unchanged
   d. useMemo/useCallback → stabilize references for memo

4. Measure again
   Confirm the fix actually helped
   Don't guess — re-renders that feel slow are rarely the bottleneck
```
