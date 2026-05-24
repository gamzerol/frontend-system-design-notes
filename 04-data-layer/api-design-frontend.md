# API Design for Frontend

Frontend engineers don't just consume APIs — they shape them. The choice between REST, GraphQL, and tRPC has real consequences for bundle size, developer experience, and how much data travels over the wire. The BFF (Backend for Frontend) pattern solves the mismatch between what backends expose and what UIs actually need.

---

## REST vs GraphQL vs tRPC

Three fundamentally different approaches to the same problem: getting data from server to client.

```
┌──────────────────────────────────────────────────────────────────┐
│                    API Approach Comparison                       │
│                                                                  │
│  REST              GraphQL             tRPC                      │
│  ──────            ───────             ────                      │
│  Multiple URLs     Single endpoint     Function calls            │
│  /products         /graphql            products.list()           │
│  /products/:id                         products.byId()           │
│  /orders                                                         │
│                                                                  │
│  Server defines    Client defines      Types shared              │
│  response shape    response shape      end-to-end                │
│                                                                  │
│  Over/under-fetch  Precise fetch       Precise fetch             │
│  is common         always              always                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## REST

The default. Resources are URLs, operations are HTTP verbs.

```
GET    /products           → list products
GET    /products/:id       → get one product
POST   /products           → create product
PATCH  /products/:id       → update product
DELETE /products/:id       → delete product
```

### The Over-fetching Problem

The server decides what's in the response. The UI often gets more than it needs:

```typescript
// UI only needs: id, name, price, imageUrl
// API returns:
{
  "id": "p1",
  "name": "Keyboard",
  "price": 129.99,
  "imageUrl": "...",
  "description": "...",        // not needed
  "sku": "KB-001",             // not needed
  "weight": 0.8,               // not needed
  "dimensions": {...},         // not needed
  "supplier": {...},           // not needed
  "inventory": {...},          // not needed
  "createdAt": "...",          // not needed
  "updatedAt": "..."           // not needed
}
```

Over-fetching wastes bandwidth, especially on mobile. The fix is either a BFF layer or GraphQL.

### The Under-fetching Problem

One endpoint doesn't return enough — you need multiple requests:

```typescript
// Render a product page: need product + reviews + related products
const product = await fetch(`/products/${id}`);
const reviews = await fetch(`/products/${id}/reviews`); // waterfall
const related = await fetch(`/products/${id}/related`); // waterfall

// Three sequential requests, each waiting for the previous
// Total time = T(product) + T(reviews) + T(related)
```

The fix: parallel fetching with `Promise.all`, or a dedicated endpoint that returns all three.

### REST Best Practices from the Frontend

```typescript
// shared/lib/httpClient.ts — centralized REST client
class HttpClient {
  private baseUrl: string;
  private defaultHeaders: Record<string, string>;

  constructor(baseUrl: string) {
    this.baseUrl = baseUrl;
    this.defaultHeaders = {
      "Content-Type": "application/json",
    };
  }

  private async request<T>(url: string, options: RequestInit = {}): Promise<T> {
    const token = tokenStore.get();

    const response = await fetch(`${this.baseUrl}${url}`, {
      ...options,
      headers: {
        ...this.defaultHeaders,
        ...(token ? { Authorization: `Bearer ${token}` } : {}),
        ...options.headers,
      },
    });

    if (response.status === 401) {
      await tokenStore.refresh(); // try to refresh token
      return this.request(url, options); // retry once
    }

    if (!response.ok) {
      const error = await response.json().catch(() => ({}));
      throw new ApiError(response.status, error.message ?? "Unknown error");
    }

    if (response.status === 204) return undefined as T;
    return response.json();
  }

  get = <T>(url: string) => this.request<T>(url);
  post = <T>(url: string, body: unknown) =>
    this.request<T>(url, { method: "POST", body: JSON.stringify(body) });
  patch = <T>(url: string, body: unknown) =>
    this.request<T>(url, { method: "PATCH", body: JSON.stringify(body) });
  delete = <T>(url: string) => this.request<T>(url, { method: "DELETE" });
}

export const httpClient = new HttpClient(import.meta.env.VITE_API_URL);
```

---

## GraphQL

One endpoint, the client defines exactly what it needs.

```graphql
# Client asks for exactly what it needs — no over/under-fetching
query GetProductPage($id: ID!) {
  product(id: $id) {
    id
    name
    price
    imageUrl
    reviews(first: 5) {
      # nested resources in one request
      rating
      comment
      author {
        name
      }
    }
    related(first: 4) {
      id
      name
      price
      imageUrl
    }
  }
}
```

One request. Exactly the fields needed. Nested resources included.

### GraphQL in React with Apollo Client

```tsx
// features/products/hooks/useProductPage.ts
import { useQuery, gql } from "@apollo/client";

const GET_PRODUCT_PAGE = gql`
  query GetProductPage($id: ID!) {
    product(id: $id) {
      id
      name
      price
      imageUrl
      reviews(first: 5) {
        rating
        comment
        author {
          name
        }
      }
      related(first: 4) {
        id
        name
        price
        imageUrl
      }
    }
  }
`;

export function useProductPage(id: string) {
  return useQuery(GET_PRODUCT_PAGE, {
    variables: { id },
    skip: !id,
  });
}

// Component
function ProductPage({ id }: { id: string }) {
  const { data, loading, error } = useProductPage(id);

  if (loading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;

  const { product } = data;
  return (
    <>
      <ProductDetail product={product} />
      <ReviewList reviews={product.reviews} />
      <RelatedProducts products={product.related} />
    </>
  );
}
```

### GraphQL Trade-offs

```
Advantages:
  ✓ No over/under-fetching
  ✓ Single request for complex pages
  ✓ Self-documenting schema
  ✓ Strongly typed (with codegen)

Disadvantages:
  ✗ Complexity: schema, resolvers, caching strategy
  ✗ HTTP caching doesn't work out of the box (POST requests)
  ✗ N+1 query problem on the server (DataLoader required)
  ✗ Overkill for simple CRUD apps
  ✗ Large bundle: Apollo Client ~32kb gzipped
```

---

## tRPC

Type-safe API calls that feel like local function calls. No schema definition, no code generation — types flow automatically from server to client.

```typescript
// server/router.ts — define procedures
import { z } from "zod";
import { router, publicProcedure } from "./trpc";

export const productRouter = router({
  list: publicProcedure
    .input(z.object({ category: z.string().optional() }))
    .query(async ({ input }) => {
      return db.products.findMany({ where: { category: input.category } });
    }),

  byId: publicProcedure.input(z.string()).query(async ({ input: id }) => {
    return db.products.findUnique({ where: { id } });
  }),

  create: publicProcedure
    .input(z.object({ name: z.string(), price: z.number() }))
    .mutation(async ({ input }) => {
      return db.products.create({ data: input });
    }),
});
```

```typescript
// client — call server functions with full TypeScript types
// No fetch(), no URL strings, no response parsing

const { data: products } = trpc.product.list.useQuery({ category: "shoes" });

const { data: product } = trpc.product.byId.useQuery(productId);

const createProduct = trpc.product.create.useMutation({
  onSuccess: () => utils.product.list.invalidate(),
});

// TypeScript knows the return type of every call — automatically
// If the server changes a field name, the client gets a type error immediately
```

### tRPC Trade-offs

```
Advantages:
  ✓ End-to-end type safety with zero boilerplate
  ✓ No code generation step
  ✓ Excellent DX: autocomplete, refactoring works across the stack
  ✓ Built on React Query — all the caching benefits included

Disadvantages:
  ✗ Only works when you own both frontend and backend (TypeScript)
  ✗ Not suitable for public APIs or third-party consumers
  ✗ Tight frontend/backend coupling (different repo = hard to use)
```

---

## The BFF Pattern (Backend for Frontend)

A BFF is a dedicated backend layer that sits between the frontend and the underlying microservices. It exists to serve the UI's exact needs.

```
Without BFF:

Frontend ──► Product Service   (returns too much data)
         ──► Review Service    (separate request)
         ──► Inventory Service (separate request)
         ──► Pricing Service   (separate request)

Problems:
  - 4 requests for one page
  - Frontend must aggregate and transform data
  - Over-fetching from each service
  - Auth/error handling duplicated in each call


With BFF:

Frontend ──► BFF ──► Product Service
                ──► Review Service
                ──► Inventory Service
                ──► Pricing Service

BFF aggregates, transforms, and returns exactly what the UI needs
in a single response:
{
  product: { id, name, price, imageUrl },
  reviews: [{ rating, comment }],
  inStock: true,
  discountedPrice: 109.99
}
```

```
┌──────────────────────────────────────────────────────────────────┐
│                      BFF Architecture                            │
│                                                                  │
│   Browser                                                        │
│      │                                                           │
│      │  GET /api/product-page/:id                                │
│      ▼                                                           │
│   ┌──────────────────────────┐                                   │
│   │     BFF (Node.js)        │                                   │
│   │  /api/product-page/:id   │                                   │
│   │                          │                                   │
│   │  Parallel fetch:         │                                   │
│   │  Promise.all([           │──► Product Service                │
│   │    getProduct(id),       │──► Review Service                 │
│   │    getReviews(id),       │──► Inventory Service              │
│   │    getInventory(id)      │                                   │
│   │  ])                      │                                   │
│   │                          │                                   │
│   │  Transform & aggregate   │                                   │
│   │  Return shaped response  │                                   │
│   └──────────────────────────┘                                   │
│      │                                                           │
│      │  { product, reviews, inStock }                            │
│      ▼                                                           │
│   Browser renders immediately with all data                      │
└──────────────────────────────────────────────────────────────────┘
```

### When to Use a BFF

```
Use a BFF when:
  ✓ Multiple microservices need to be aggregated for one page
  ✓ You need to transform/reshape data for the UI
  ✓ You want to move auth token handling server-side
  ✓ You have different clients (web, mobile) with different needs
     (web BFF vs mobile BFF — each returns what its client needs)

Skip a BFF when:
  ✗ You have a monolithic backend that already returns shaped data
  ✗ Your team doesn't own the backend
  ✗ The overhead of maintaining another service isn't justified
```

---

## API Versioning Strategies

APIs change. Strategies for handling breaking changes from the frontend:

**1. URL versioning** — most common

```
/api/v1/products   ← old clients use this
/api/v2/products   ← new clients use this
```

Simple, explicit, cacheable. Downside: proliferates endpoints.

**2. Header versioning**

```typescript
headers: { 'API-Version': '2024-01-01' }
```

Cleaner URLs. Harder to test in a browser.

**3. Consumer-driven contracts** (advanced)
Frontend defines what it expects. Backend tests against those contracts before shipping.

**Frontend strategy:** always code defensively against API changes:

```typescript
// ❌ Fragile — breaks if field is missing or renamed
const price = product.pricing.salePrice;

// ✅ Defensive — handle missing fields gracefully
const price = product?.pricing?.salePrice ?? product?.price ?? 0;

// ✅ Transform at the boundary — isolate the API shape in the API layer
// If the API changes, you update one place: the API function
function toProduct(raw: RawProductDTO): Product {
  return {
    id: raw.id,
    name: raw.name,
    price: raw.pricing?.salePrice ?? raw.price ?? 0,
    imageUrl: raw.images?.[0]?.url ?? "",
  };
}
```

---

## Comparison Table

|                    | REST                  | GraphQL             | tRPC                         |
| ------------------ | --------------------- | ------------------- | ---------------------------- |
| **Learning curve** | Low                   | High                | Medium                       |
| **Type safety**    | Manual                | With codegen        | Automatic                    |
| **Over-fetching**  | Common                | None                | None                         |
| **Bundle size**    | ~0 (fetch)            | ~32kb (Apollo)      | ~45kb (includes React Query) |
| **HTTP caching**   | ✓                     | ✗ (needs config)    | ✓                            |
| **Public API**     | ✓                     | ✓                   | ✗                            |
| **Full-stack TS**  | Optional              | Optional            | Required                     |
| **Best for**       | Any team, any backend | Complex data graphs | Full-stack TS teams          |
