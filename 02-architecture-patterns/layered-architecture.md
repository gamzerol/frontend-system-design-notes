# Layered Architecture in Frontend

Layered architecture organizes code into horizontal layers — each layer has a single responsibility and can only communicate with the layer directly below it. In frontend, the three core layers are: **Presentation**, **Business Logic**, and **Data Access**. This separation makes code testable, replaceable, and easier to reason about.

---

## Why Layers?

Without layers, components do everything:

```tsx
// ❌ No layers — component does everything
function OrderList() {
  const [orders, setOrders] = useState([]);

  useEffect(() => {
    // Data fetching mixed with component logic
    fetch("/api/orders")
      .then((res) => res.json())
      .then((data) => {
        // Business logic mixed in
        const filtered = data.filter((o) => o.status !== "cancelled");
        const sorted = filtered.sort((a, b) => b.createdAt - a.createdAt);
        setOrders(sorted);
      });
  }, []);

  // Formatting logic mixed in
  const formatDate = (ts) => new Date(ts).toLocaleDateString("en-US");

  return (
    <ul>
      {orders.map((order) => (
        <li key={order.id}>
          {order.id} — {formatDate(order.createdAt)}
        </li>
      ))}
    </ul>
  );
}
```

Problems:

- Can't test the filtering/sorting logic without rendering the component
- Can't swap the API endpoint without touching the component
- Can't reuse the formatting logic elsewhere
- One change can break three unrelated concerns

---

## The Three Layers

```
┌─────────────────────────────────────────────────────┐
│               Presentation Layer                    │
│                                                     │
│   React components, JSX, styling, user events       │
│   What things look like and how users interact      │
│                                                     │
│   Components, Pages, Layouts, UI primitives         │
└───────────────────────┬─────────────────────────────┘
                        │ uses
                        ▼
┌─────────────────────────────────────────────────────┐
│             Business Logic Layer                    │
│                                                     │
│   Custom hooks, services, pure functions            │
│   Rules, transformations, decisions                 │
│                                                     │
│   useOrders(), OrderService, formatters             │
└───────────────────────┬─────────────────────────────┘
                        │ uses
                        ▼
┌─────────────────────────────────────────────────────┐
│               Data Access Layer                     │
│                                                     │
│   API clients, fetch wrappers, cache adapters       │
│   How data gets in and out of the app               │
│                                                     │
│   ordersApi, httpClient, localStorageAdapter        │
└─────────────────────────────────────────────────────┘
```

**The rule:** dependencies flow **downward only**. The presentation layer knows about business logic, but business logic knows nothing about React components. The data layer knows nothing about either.

---

## Layer 1: Data Access

Responsible for: talking to the outside world.

```typescript
// features/orders/api/ordersApi.ts

import { httpClient } from "@/shared/lib/httpClient";
import type { Order } from "../types";

export const ordersApi = {
  getAll: (): Promise<Order[]> => httpClient.get("/orders"),

  getById: (id: string): Promise<Order> => httpClient.get(`/orders/${id}`),

  cancel: (id: string): Promise<void> =>
    httpClient.patch(`/orders/${id}`, { status: "cancelled" }),
};
```

```typescript
// shared/lib/httpClient.ts
// One place to configure base URL, auth headers, error handling

const BASE_URL = import.meta.env.VITE_API_URL;

async function request<T>(url: string, options?: RequestInit): Promise<T> {
  const response = await fetch(`${BASE_URL}${url}`, {
    ...options,
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${getToken()}`,
      ...options?.headers,
    },
  });

  if (!response.ok) {
    throw new ApiError(response.status, await response.json());
  }

  return response.json();
}

export const httpClient = {
  get: <T>(url: string) => request<T>(url),
  post: <T>(url: string, body: unknown) =>
    request<T>(url, { method: "POST", body: JSON.stringify(body) }),
  patch: <T>(url: string, body: unknown) =>
    request<T>(url, { method: "PATCH", body: JSON.stringify(body) }),
};
```

The data layer has no knowledge of React, state, or UI. It's pure TypeScript.

---

## Layer 2: Business Logic

Responsible for: transformations, rules, decisions.

In React apps, this layer lives in **custom hooks** (stateful) and **pure functions** (stateless).

```typescript
// features/orders/hooks/useOrders.ts
// Stateful business logic — uses React Query + applies rules

import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { ordersApi } from "../api/ordersApi";
import { filterActiveOrders, sortByDate } from "../lib/orderUtils";

export function useOrders() {
  const { data, isLoading, error } = useQuery({
    queryKey: ["orders"],
    queryFn: ordersApi.getAll,
    select: (data) => sortByDate(filterActiveOrders(data)), // transform here
  });

  return { orders: data ?? [], isLoading, error };
}

export function useCancelOrder() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ordersApi.cancel,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["orders"] });
    },
  });
}
```

```typescript
// features/orders/lib/orderUtils.ts
// Pure functions — stateless, easily testable

import type { Order } from "../types";

export function filterActiveOrders(orders: Order[]): Order[] {
  return orders.filter((o) => o.status !== "cancelled");
}

export function sortByDate(orders: Order[]): Order[] {
  return [...orders].sort(
    (a, b) => new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime(),
  );
}

export function formatOrderDate(isoString: string): string {
  return new Intl.DateTimeFormat("en-US", {
    year: "numeric",
    month: "short",
    day: "numeric",
  }).format(new Date(isoString));
}
```

Pure functions are the easiest things to test — no mocks, no setup, just input → output.

---

## Layer 3: Presentation

Responsible for: rendering UI, handling user events.

```tsx
// features/orders/components/OrderList.tsx
// Component knows nothing about fetch, filtering, or sorting

import { useOrders, useCancelOrder } from "../hooks/useOrders";
import { formatOrderDate } from "../lib/orderUtils";
import { OrderCard } from "./OrderCard";
import { Spinner } from "@/shared/ui/Spinner";
import { ErrorMessage } from "@/shared/ui/ErrorMessage";

export function OrderList() {
  const { orders, isLoading, error } = useOrders();
  const cancelOrder = useCancelOrder();

  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;

  return (
    <ul>
      {orders.map((order) => (
        <OrderCard
          key={order.id}
          order={order}
          formattedDate={formatOrderDate(order.createdAt)}
          onCancel={() => cancelOrder.mutate(order.id)}
        />
      ))}
    </ul>
  );
}
```

```tsx
// features/orders/components/OrderCard.tsx
// Pure presentational component — no hooks, no side effects

type Props = {
  order: Order;
  formattedDate: string;
  onCancel: () => void;
};

export function OrderCard({ order, formattedDate, onCancel }: Props) {
  return (
    <li>
      <span>{order.id}</span>
      <span>{formattedDate}</span>
      <span>{order.status}</span>
      <button onClick={onCancel}>Cancel</button>
    </li>
  );
}
```

`OrderCard` is a **pure presentational component** — it receives everything via props and has no side effects. It's trivial to test and reuse.

---

## The Dependency Rule in Practice

```
┌──────────────────────────────────────────────────────┐
│  Presentation    OrderList.tsx, OrderCard.tsx        │
│  knows about: →  hooks, formatters, types            │
├──────────────────────────────────────────────────────┤
│  Business Logic  useOrders.ts, orderUtils.ts         │
│  knows about: →  ordersApi, types                    │
├──────────────────────────────────────────────────────┤
│  Data Access     ordersApi.ts, httpClient.ts         │
│  knows about: →  types, environment variables        │
└──────────────────────────────────────────────────────┘

✅ OrderList imports useOrders         (presentation → logic)
✅ useOrders imports ordersApi         (logic → data)
❌ ordersApi imports useOrders         (data → logic: violation)
❌ orderUtils imports OrderCard        (logic → presentation: violation)
```

---

## Folder Structure

```
features/orders/
├── api/
│   └── ordersApi.ts           ← Data Access Layer
│
├── hooks/
│   └── useOrders.ts           ← Business Logic (stateful)
│
├── lib/
│   └── orderUtils.ts          ← Business Logic (pure functions)
│
├── components/
│   ├── OrderList.tsx          ← Presentation (smart, uses hooks)
│   └── OrderCard.tsx          ← Presentation (dumb, props only)
│
├── types/
│   └── index.ts               ← Shared types for this feature
│
└── index.ts                   ← Public API of this feature
```

---

## Testing Each Layer

Layered architecture makes testing dramatically easier:

```typescript
// ✅ Data layer — mock fetch, test API shapes
it('fetches and returns orders', async () => {
  global.fetch = vi.fn().mockResolvedValue({
    ok: true,
    json: () => Promise.resolve([{ id: 'o1', status: 'active' }]),
  });

  const orders = await ordersApi.getAll();
  expect(orders).toHaveLength(1);
});

// ✅ Business logic — no mocks needed, pure functions
it('filters out cancelled orders', () => {
  const orders = [
    { id: 'o1', status: 'active' },
    { id: 'o2', status: 'cancelled' },
  ];
  expect(filterActiveOrders(orders)).toEqual([{ id: 'o1', status: 'active' }]);
});

// ✅ Presentation — mock the hook, test rendering
it('renders an order for each item', () => {
  vi.mocked(useOrders).mockReturnValue({
    orders: [{ id: 'o1', status: 'active', createdAt: '2024-01-01' }],
    isLoading: false,
    error: null,
  });

  render(<OrderList />);
  expect(screen.getByText('o1')).toBeInTheDocument();
});
```

Each layer is tested in isolation. No test requires setting up the entire system.

---

## Trade-offs

|                      | Layered                                 | No Layers                     |
| -------------------- | --------------------------------------- | ----------------------------- |
| **Testability**      | High — each layer testable in isolation | Low — everything coupled      |
| **Replaceability**   | Swap API client without touching UI     | Hard — everything intertwined |
| **Boilerplate**      | More files, more indirection            | Less files                    |
| **Learning curve**   | Requires team alignment                 | Easier to start               |
| **Scales with team** | Well                                    | Poorly                        |

> **For small apps (1–2 engineers, few features):** layered architecture may be overkill. Start simpler, introduce layers when the pain of not having them becomes real.
