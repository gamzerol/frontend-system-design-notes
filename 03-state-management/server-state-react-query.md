# Server State & React Query

Server state is data that lives on the server and is only cached temporarily on the client. It's fundamentally different from client state — it can go stale, needs background refetching, and can be changed by other users. React Query (TanStack Query) is purpose-built for server state: it handles caching, background sync, loading/error states, and cache invalidation so you don't have to build it yourself.

---

## Client State vs Server State

This distinction is the most important concept in modern React data management:

```
┌─────────────────────────────────────────────────────────────────┐
│                Client State vs Server State                     │
│                                                                 │
│  CLIENT STATE                    SERVER STATE                   │
│  ─────────────                   ────────────                   │
│  • Lives in the browser          • Lives on the server          │
│  • You own it                    • You borrow a snapshot of it  │
│  • Never goes stale              • Goes stale immediately       │
│  • Changed only by this user     • Changed by anyone            │
│  • Lost on refresh               • Persisted in a database      │
│                                                                 │
│  Examples:                       Examples:                      │
│  isOpen, theme, selected tab     Products, orders, user profile │
│  form draft, notification pref   Comments, inventory, prices    │
│                                                                 │
│  Tool: useState, Zustand         Tool: React Query, SWR         │
└─────────────────────────────────────────────────────────────────┘
```

The mistake most apps make: putting server state into Redux or Zustand. Now you're responsible for fetching, caching, invalidating, and syncing data that the server already manages. React Query does all of this automatically.

---

## What React Query Solves

Without React Query, this is what a "simple" data fetch looks like:

```tsx
// ❌ Manual data fetching — you manage everything
function ProductList() {
  const [products, setProducts] = useState([]);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    setIsLoading(true);
    fetch("/api/products")
      .then((res) => res.json())
      .then((data) => {
        setProducts(data);
        setIsLoading(false);
      })
      .catch((err) => {
        setError(err);
        setIsLoading(false);
      });
  }, []);

  // Problems this doesn't handle:
  // - No caching: fetches on every mount
  // - No deduplication: two components fetch the same data simultaneously
  // - No background refetch: stale data shown until manual refresh
  // - No retry on failure
  // - No loading state between refetches (data flickers to empty)
  // - Race conditions on rapid re-renders
}
```

With React Query:

```tsx
// ✅ React Query handles all of the above
function ProductList() {
  const {
    data: products,
    isLoading,
    error,
  } = useQuery({
    queryKey: ["products"],
    queryFn: () => fetch("/api/products").then((res) => res.json()),
  });

  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return (
    <ul>
      {products.map((p) => (
        <ProductCard key={p.id} product={p} />
      ))}
    </ul>
  );
}
```

---

## The Cache Lifecycle

React Query's cache is the core of how it works. Every query has a lifecycle:

```
┌──────────────────────────────────────────────────────────────────┐
│                    React Query Cache Lifecycle                   │
│                                                                  │
│  useQuery called                                                 │
│       │                                                          │
│       ▼                                                          │
│  Cache hit?                                                      │
│  ├── NO  → fetch immediately → store in cache → serve data       │
│  └── YES → serve cached data immediately (no loading spinner)    │
│             │                                                    │
│             ▼                                                    │
│        Is it stale? (older than staleTime)                       │
│        ├── NO  → show cached data, done                          │
│        └── YES → show cached data AND refetch in background      │
│                   └── when fetch completes → update UI silently  │
│                                                                  │
│  Component unmounts                                              │
│       │                                                          │
│       ▼                                                          │
│  Query becomes inactive                                          │
│  After gcTime (default 5min) → removed from cache                │
└──────────────────────────────────────────────────────────────────┘
```

The key insight: **stale data is shown immediately** while fresh data is fetched in the background. The user never sees a loading spinner on revisit — they see the last known data, then it updates silently.

---

## Key Configuration Options

```typescript
const { data } = useQuery({
  queryKey: ["products", { category: "shoes" }],
  queryFn: () => productsApi.getByCategory("shoes"),

  staleTime: 1000 * 60 * 5, // 5 minutes — how long data is "fresh"
  // During this time, no background refetch
  // Default: 0 (always stale)

  gcTime: 1000 * 60 * 10, // 10 minutes — how long inactive cache is kept
  // After component unmounts, cache lives this long
  // Default: 5 minutes

  refetchOnWindowFocus: true, // Refetch when user returns to tab
  // Great for data that changes often

  refetchInterval: 1000 * 30, // Poll every 30 seconds (for live data)

  retry: 3, // Retry failed requests 3 times
  retryDelay: (
    attemptIndex, // Exponential backoff
  ) => Math.min(1000 * 2 ** attemptIndex, 30000),

  enabled: !!userId, // Only fetch when userId is defined
  // Great for dependent queries
});
```

### staleTime Strategy by Data Type

```
staleTime: 0          (default)
  → User profile shown in a header, notifications, cart
  → Always refetch in background on component mount

staleTime: 60_000     (1 minute)
  → Product listings, search results
  → Mostly stable but can change

staleTime: 300_000    (5 minutes)
  → Category list, config data, feature flags
  → Changes rarely

staleTime: Infinity
  → Static reference data (countries, currencies, enums)
  → Fetch once, never refetch
```

---

## Query Keys

Query keys are how React Query identifies and deduplicates cache entries. Treat them like a cache key — they must uniquely identify the data.

```typescript
// Simple key — all products
useQuery({ queryKey: ['products'], queryFn: ... })

// Parameterized key — products filtered by category
useQuery({ queryKey: ['products', { category }], queryFn: ... })

// Nested resource
useQuery({ queryKey: ['products', productId], queryFn: ... })
useQuery({ queryKey: ['products', productId, 'reviews'], queryFn: ... })

// Key changes → new fetch
// Key is the same → cached result is returned
```

Convention: structure keys hierarchically. This lets you invalidate at any level:

```typescript
// Invalidate all product queries
queryClient.invalidateQueries({ queryKey: ["products"] });

// Invalidate only a specific product
queryClient.invalidateQueries({ queryKey: ["products", productId] });
```

---

## Mutations: Writing Data

`useMutation` handles create, update, and delete operations:

```typescript
// features/products/hooks/useCreateProduct.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';

export function useCreateProduct() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (newProduct: CreateProductDTO) =>
      productsApi.create(newProduct),

    onSuccess: (createdProduct) => {
      // Invalidate product list so it refetches with the new item
      queryClient.invalidateQueries({ queryKey: ['products'] });

      // Or directly update the cache (faster — no extra network request)
      queryClient.setQueryData(
        ['products', createdProduct.id],
        createdProduct
      );
    },

    onError: (error) => {
      toast.error(`Failed to create product: ${error.message}`);
    },
  });
}

// Usage
function CreateProductForm() {
  const createProduct = useCreateProduct();

  function handleSubmit(data: CreateProductDTO) {
    createProduct.mutate(data);
  }

  return (
    <form onSubmit={handleSubmit}>
      {/* ... */}
      <button type="submit" disabled={createProduct.isPending}>
        {createProduct.isPending ? 'Creating...' : 'Create'}
      </button>
    </form>
  );
}
```

---

## Optimistic Updates

Show the result of an action immediately, before the server confirms it. If the request fails, roll back.

```typescript
export function useToggleLike(postId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: () => postsApi.toggleLike(postId),

    // 1. Before the request — optimistically update the cache
    onMutate: async () => {
      // Cancel any in-flight refetches (avoid overwriting our optimistic update)
      await queryClient.cancelQueries({ queryKey: ["posts", postId] });

      // Snapshot current value for rollback
      const previousPost = queryClient.getQueryData(["posts", postId]);

      // Optimistically update
      queryClient.setQueryData(["posts", postId], (old: Post) => ({
        ...old,
        isLiked: !old.isLiked,
        likeCount: old.isLiked ? old.likeCount - 1 : old.likeCount + 1,
      }));

      return { previousPost }; // returned as context
    },

    // 2. On error — roll back to the snapshot
    onError: (err, variables, context) => {
      queryClient.setQueryData(["posts", postId], context?.previousPost);
      toast.error("Failed to update like");
    },

    // 3. On settle (success or error) — sync with server truth
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ["posts", postId] });
    },
  });
}
```

The pattern: **snapshot → optimistic update → rollback on error → sync on settle**.

---

## Dependent Queries

Sometimes a query depends on the result of another:

```typescript
// Step 1: fetch the user
const { data: user } = useQuery({
  queryKey: ["user"],
  queryFn: authApi.getUser,
});

// Step 2: fetch orders only when userId is available
const { data: orders } = useQuery({
  queryKey: ["orders", user?.id],
  queryFn: () => ordersApi.getByUser(user!.id),
  enabled: !!user?.id, // ← query won't run until this is true
});
```

---

## Prefetching

Load data before the user navigates — so it's ready when they arrive:

```typescript
const queryClient = useQueryClient();

// On hover — prefetch the product detail before click
function ProductCard({ product }) {
  function handleMouseEnter() {
    queryClient.prefetchQuery({
      queryKey: ['products', product.id],
      queryFn: () => productsApi.getById(product.id),
      staleTime: 60_000,
    });
  }

  return (
    <div onMouseEnter={handleMouseEnter}>
      {product.name}
    </div>
  );
}
```

When the user clicks and navigates to the product detail page, the data is already in cache. No loading spinner.

---

## Structuring React Query in a Feature

```typescript
// features/products/api/productsApi.ts  ← pure API functions
export const productsApi = {
  getAll: (): Promise<Product[]> => httpClient.get("/products"),
  getById: (id: string): Promise<Product> => httpClient.get(`/products/${id}`),
  create: (dto: CreateProductDTO): Promise<Product> =>
    httpClient.post("/products", dto),
};

// features/products/hooks/useProducts.ts  ← React Query hooks
export function useProducts() {
  return useQuery({
    queryKey: ["products"],
    queryFn: productsApi.getAll,
  });
}

export function useProduct(id: string) {
  return useQuery({
    queryKey: ["products", id],
    queryFn: () => productsApi.getById(id),
    enabled: !!id,
  });
}

// features/products/components/ProductList.tsx  ← component stays clean
export function ProductList() {
  const { data, isLoading, error } = useProducts();
  // ...
}
```

Components never call `fetch` directly. The hook layer handles all React Query config.

---

## React Query vs Redux for Server State

A common question: "Should I use Redux or React Query for my API data?"

```
With Redux:
  ├── Write a thunk/saga to fetch data
  ├── Handle loading, error states in the reducer
  ├── Write selectors to read data
  ├── Manually invalidate cache on mutation
  ├── Handle deduplication yourself
  └── Background refetch? Write that too.

With React Query:
  └── Write queryFn
      Everything else is handled.
```

Redux is not designed for server state — it has no concept of staleness, background sync, or cache TTL. Using Redux for server data means reimplementing what React Query already does, but worse.

**Use both together:** React Query for server state, Zustand for client state. They don't conflict — they complement each other.
