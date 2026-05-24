# Error Handling

Error handling is a system design concern, not an afterthought. A well-designed error handling strategy classifies errors by type, handles them at the right layer, shows users the right message, and gives engineers the right information for debugging. The goal is to never show a user a blank screen or a raw stack trace — and to never lose a bug silently.

---

## Error Classification

Not all errors are the same. Treating them differently is the foundation of good error handling:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Error Classification                         │
│                                                                 │
│  Network Errors          Validation Errors                      │
│  ────────────────         ──────────────────                    │
│  No internet             Form field invalid                     │
│  Server unreachable      Missing required field                 │
│  Timeout                 Value out of range                     │
│  → Retry or offline UI   → Show inline field errors             │
│                                                                 │
│  Business Errors         Unexpected Errors                      │
│  ───────────────         ─────────────────                      │
│  Not authorized          JavaScript crash                       │
│  Item out of stock       Null reference                         │
│  Payment declined        Unhandled promise                      │
│  → Show specific message → Error boundary + log to Sentry       │
└─────────────────────────────────────────────────────────────────┘
```

Each type has a different cause, a different user message, and a different recovery path. Lumping them together leads to generic "Something went wrong" messages that help nobody.

---

## Custom Error Classes

Model errors as typed objects, not strings:

```typescript
// shared/lib/errors.ts

export class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly statusCode?: number,
  ) {
    super(message);
    this.name = "AppError";
  }
}

export class NetworkError extends AppError {
  constructor(message = "Network unavailable") {
    super(message, "NETWORK_ERROR");
    this.name = "NetworkError";
  }
}

export class ApiError extends AppError {
  constructor(
    public readonly statusCode: number,
    message: string,
    public readonly field?: string,
  ) {
    super(message, "API_ERROR", statusCode);
    this.name = "ApiError";
  }
}

export class ValidationError extends AppError {
  constructor(
    message: string,
    public readonly fields: Record<string, string>,
  ) {
    super(message, "VALIDATION_ERROR");
    this.name = "ValidationError";
  }
}

// Type guard — check error type safely
export function isApiError(error: unknown): error is ApiError {
  return error instanceof ApiError;
}

export function isNetworkError(error: unknown): error is NetworkError {
  return error instanceof NetworkError;
}
```

---

## Layer 1: HTTP Client — Normalize Errors at the Boundary

All API errors should be caught and normalized at the HTTP client level. Components and hooks should never see raw fetch errors.

```typescript
// shared/lib/httpClient.ts

async function request<T>(url: string, options?: RequestInit): Promise<T> {
  // Network-level errors (no connection, DNS failure, timeout)
  let response: Response;
  try {
    response = await fetch(url, options);
  } catch (err) {
    throw new NetworkError(
      "Failed to reach the server. Check your connection.",
    );
  }

  // HTTP error responses (4xx, 5xx)
  if (!response.ok) {
    let body: { message?: string; errors?: Record<string, string> } = {};

    try {
      body = await response.json();
    } catch {
      // Response body isn't JSON — that's okay
    }

    // Validation errors from the server (422 Unprocessable Entity)
    if (response.status === 422 && body.errors) {
      throw new ValidationError(
        body.message ?? "Validation failed",
        body.errors,
      );
    }

    // All other HTTP errors
    throw new ApiError(
      response.status,
      body.message ?? `Request failed with status ${response.status}`,
    );
  }

  if (response.status === 204) return undefined as T;
  return response.json();
}
```

Now the rest of the app works with typed, predictable errors instead of raw fetch failures.

---

## Layer 2: React Query — Per-Query Error Handling

React Query surfaces errors from the query function. Handle them in hooks:

```typescript
// features/orders/hooks/useOrders.ts

export function useOrders() {
  return useQuery({
    queryKey: ['orders'],
    queryFn: ordersApi.getAll,
    retry: (failureCount, error) => {
      // Don't retry 4xx errors — they won't succeed
      if (isApiError(error) && error.statusCode < 500) return false;
      // Retry network errors and 5xx up to 3 times
      return failureCount < 3;
    },
  });
}

// Component handles the error state
function OrderList() {
  const { data, isLoading, error } = useOrders();

  if (isLoading) return <Spinner />;

  if (error) {
    if (isNetworkError(error)) {
      return <OfflineBanner onRetry={() => refetch()} />;
    }
    if (isApiError(error) && error.statusCode === 403) {
      return <AccessDenied />;
    }
    return <ErrorMessage message="Could not load orders. Please try again." />;
  }

  return <ul>{data.map(order => <OrderCard key={order.id} order={order} />)}</ul>;
}
```

---

## Layer 3: Error Boundaries — Catch Unexpected React Errors

Error boundaries catch JavaScript errors thrown during rendering. They prevent the entire app from crashing.

```tsx
// shared/ui/ErrorBoundary.tsx

type Props = {
  fallback: ReactNode | ((error: Error, reset: () => void) => ReactNode);
  children: ReactNode;
  onError?: (error: Error, info: ErrorInfo) => void;
};

type State = { error: Error | null };

export class ErrorBoundary extends Component<Props, State> {
  state: State = { error: null };

  static getDerivedStateFromError(error: Error): State {
    return { error };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    // Log to error monitoring (Sentry, Datadog, etc.)
    this.props.onError?.(error, info);
    errorMonitor.capture(error, {
      componentStack: info.componentStack,
    });
  }

  reset = () => this.setState({ error: null });

  render() {
    if (this.state.error) {
      const { fallback } = this.props;
      return typeof fallback === "function"
        ? fallback(this.state.error, this.reset)
        : fallback;
    }
    return this.props.children;
  }
}
```

### Placement Strategy

Don't put one error boundary at the root and call it done. Place them at feature boundaries:

```tsx
// app/App.tsx — root boundary catches catastrophic failures
<ErrorBoundary fallback={<AppCrashPage />}>
  <Router>
    {/* feature-level boundaries — isolate failures */}
    <Route
      path="/products"
      element={
        <ErrorBoundary
          fallback={(error, reset) => (
            <FeatureError message="Products failed to load" onRetry={reset} />
          )}
        >
          <ProductsPage />
        </ErrorBoundary>
      }
    />
    <Route
      path="/cart"
      element={
        <ErrorBoundary fallback={<FeatureError message="Cart unavailable" />}>
          <CartPage />
        </ErrorBoundary>
      }
    />
  </Router>
</ErrorBoundary>
```

If Products crashes, Cart still works. The user loses one feature, not the whole app.

---

## Layer 4: Global Unhandled Error Catching

Catch errors that slip past everything else:

```typescript
// app/errorMonitoring.ts

// Unhandled promise rejections
window.addEventListener("unhandledrejection", (event) => {
  errorMonitor.capture(event.reason, {
    type: "unhandledRejection",
  });
  // Prevent the browser from logging it as an uncaught error
  event.preventDefault();
});

// Synchronous errors outside React
window.addEventListener("error", (event) => {
  errorMonitor.capture(event.error, {
    type: "globalError",
    filename: event.filename,
    lineno: event.lineno,
  });
});
```

---

## Form Validation Errors

Validation errors need special treatment — they're per-field, not global:

```tsx
// Using React Hook Form + Zod for schema validation
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const schema = z.object({
  email: z.string().email("Enter a valid email address"),
  password: z.string().min(8, "Password must be at least 8 characters"),
});

type FormData = z.infer<typeof schema>;

function LoginForm() {
  const {
    register,
    handleSubmit,
    setError,
    formState: { errors, isSubmitting },
  } = useForm<FormData>({ resolver: zodResolver(schema) });

  async function onSubmit(data: FormData) {
    try {
      await authApi.login(data);
    } catch (err) {
      if (isApiError(err) && err.statusCode === 401) {
        // Server says credentials are wrong → show on password field
        setError("password", { message: "Incorrect email or password" });
      } else if (err instanceof ValidationError) {
        // Server returned field-level errors → map to form
        Object.entries(err.fields).forEach(([field, message]) => {
          setError(field as keyof FormData, { message });
        });
      } else {
        // Unknown error → show general error
        setError("root", { message: "Login failed. Please try again." });
      }
    }
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <input {...register("email")} />
        {errors.email && <span>{errors.email.message}</span>}
      </div>
      <div>
        <input type="password" {...register("password")} />
        {errors.password && <span>{errors.password.message}</span>}
      </div>
      {errors.root && <p className="error">{errors.root.message}</p>}
      <button type="submit" disabled={isSubmitting}>
        Login
      </button>
    </form>
  );
}
```

---

## Retry Strategies

Not all errors warrant a retry. Retry the right ones, the right way:

```typescript
// Exponential backoff with jitter
function getRetryDelay(attemptIndex: number): number {
  const base = 1000; // 1 second
  const max = 30000; // 30 seconds cap
  const exponential = Math.min(base * 2 ** attemptIndex, max);
  const jitter = Math.random() * 1000; // add randomness to avoid thundering herd
  return exponential + jitter;
}

// React Query retry config
useQuery({
  queryKey: ["orders"],
  queryFn: ordersApi.getAll,
  retry: (failureCount, error) => {
    if (isApiError(error)) {
      // Never retry client errors — they need user action
      if (error.statusCode >= 400 && error.statusCode < 500) return false;
    }
    return failureCount < 3; // retry up to 3 times for server/network errors
  },
  retryDelay: getRetryDelay,
});
```

---

## User-Facing Error Messages

Good error messages tell the user what happened and what they can do:

```
❌ Bad:   "Error 500"
❌ Bad:   "Something went wrong"
❌ Bad:   "TypeError: Cannot read property 'id' of undefined"

✅ Good:  "We couldn't load your orders. Check your connection and try again."
✅ Good:  "Your card was declined. Please use a different payment method."
✅ Good:  "This item is out of stock. We'll notify you when it's available."
```

```tsx
// shared/ui/ErrorMessage.tsx — consistent error display component

type Props = {
  error: unknown;
  onRetry?: () => void;
};

export function ErrorMessage({ error, onRetry }: Props) {
  const message = getUserFriendlyMessage(error);

  return (
    <div role="alert" className="error-container">
      <p>{message}</p>
      {onRetry && <button onClick={onRetry}>Try again</button>}
    </div>
  );
}

function getUserFriendlyMessage(error: unknown): string {
  if (isNetworkError(error)) {
    return "You appear to be offline. Check your connection and try again.";
  }
  if (isApiError(error)) {
    if (error.statusCode === 403)
      return "You don't have permission to view this.";
    if (error.statusCode === 404) return "This page or item no longer exists.";
    if (error.statusCode >= 500)
      return "Our servers are having trouble. Please try again shortly.";
    return error.message; // business errors usually have a good message already
  }
  return "Something unexpected happened. Please try again.";
}
```

---

## Error Monitoring

Capturing errors in production is as important as handling them gracefully:

```typescript
// shared/lib/errorMonitor.ts — thin wrapper around Sentry

import * as Sentry from "@sentry/react";

export const errorMonitor = {
  init() {
    Sentry.init({
      dsn: import.meta.env.VITE_SENTRY_DSN,
      environment: import.meta.env.MODE,
      // Don't send errors in development
      enabled: import.meta.env.PROD,
    });
  },

  capture(error: Error, context?: Record<string, unknown>) {
    Sentry.captureException(error, { extra: context });
  },

  setUser(user: { id: string; email: string } | null) {
    Sentry.setUser(user);
  },
};
```

---

## The Full Error Handling Stack

```
User action (click, form submit, navigation)
         │
         ▼
  React component
         │
         ▼
  Custom hook (useOrders, useMutation)
         │ catches: loading/error states from React Query
         ▼
  HTTP client
         │ catches: network errors, HTTP 4xx/5xx → typed errors
         ▼
  Network / Server
         │
         ▼ (error flows back up)
  React Query
         │ retries, exposes error state
         ▼
  Component renders error UI
         │
  If rendering crashes → Error Boundary
         │ logs to Sentry, shows fallback UI
         ▼
  Global handlers (unhandledrejection, window.error)
         │ catches anything that slipped past
         ▼
  Sentry / error monitoring
```
