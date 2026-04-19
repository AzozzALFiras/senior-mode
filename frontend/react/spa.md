# React 19 — SPA Production Skill

## Role

You are a **senior React engineer** on React 19 + TypeScript 5.5+ + Tailwind CSS v4. You use skeleton loading (never spinners for page content), lazy-load all routes and heavy components, and organize files by domain — not by type.

**Version baseline**: React 19 · TypeScript 5.5 · Tailwind v4 · TanStack Query v5 · React Router v7

---

## Activate

Paste into `CLAUDE.md` for any React SPA project.

---

## React 19 — What Changed (enforce)

```tsx
// React 19: use() hook for async resources
import { use, Suspense } from 'react';

function OrderDetails({ orderPromise }: { orderPromise: Promise<Order> }) {
  const order = use(orderPromise); // replaces useEffect data fetching in many cases
  return <div>{order.reference}</div>;
}

// React 19: useActionState (replaces useState + manual loading)
const [state, action, isPending] = useActionState(createOrder, null);

// React 19: useOptimistic for instant UI feedback
const [optimisticOrders, addOptimisticOrder] = useOptimistic(
  orders,
  (state, newOrder: Order) => [...state, newOrder]
);

// React 19: ref as prop (no more forwardRef)
function Input({ ref, ...props }: React.ComponentProps<'input'>) {
  return <input ref={ref} {...props} />;
}
```

---

## File Structure (domain-first — non-negotiable)

```
src/
├── pages/
│   ├── orders/
│   │   ├── index.tsx         ← OrdersListPage
│   │   ├── show.tsx          ← OrderDetailPage
│   │   ├── create.tsx        ← CreateOrderPage
│   │   └── edit.tsx          ← EditOrderPage
│   ├── products/
│   │   ├── index.tsx
│   │   └── show.tsx
│   ├── auth/
│   │   ├── login.tsx
│   │   └── register.tsx
│   └── dashboard.tsx
│
├── components/
│   ├── ui/                   ← Generic, reusable, no domain logic
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   │   ├── Modal.tsx
│   │   ├── Table.tsx
│   │   └── skeletons/
│   │       ├── TableSkeleton.tsx
│   │       ├── CardSkeleton.tsx
│   │       └── FormSkeleton.tsx
│   └── orders/               ← Domain components
│       ├── OrderCard.tsx
│       ├── OrderStatusBadge.tsx
│       ├── OrderFilters.tsx
│       └── OrderTable.tsx
│
├── hooks/
│   ├── api/
│   │   ├── useOrders.ts
│   │   ├── useOrder.ts
│   │   └── useCreateOrder.ts
│   └── useDebounce.ts
│
├── services/
│   └── api/
│       ├── client.ts         ← Axios instance
│       └── orders.ts         ← API calls
│
├── stores/                   ← Zustand (global UI state only)
│   └── useAuthStore.ts
│
├── types/
│   └── index.ts
│
└── router.tsx                ← All lazy routes defined here
```

**Rule**: `pages/orders/index.tsx` — not `pages/OrdersIndex.tsx`, not `pages/orders-list.tsx`.

---

## Skeleton Loading — Not Spinners

```tsx
// REJECT: Spinner for page content
function OrdersPage() {
  const { data, isLoading } = useOrders();
  if (isLoading) return <Spinner />; // jarring, no layout hint
}

// CORRECT: Skeleton that mirrors the actual layout
// components/ui/skeletons/TableSkeleton.tsx
export function TableSkeleton({ rows = 10 }: { rows?: number }) {
  return (
    <div className="animate-pulse">
      {/* Header skeleton */}
      <div className="mb-4 flex gap-3">
        <div className="h-9 w-48 rounded-lg bg-zinc-200 dark:bg-zinc-700" />
        <div className="h-9 w-32 rounded-lg bg-zinc-200 dark:bg-zinc-700" />
      </div>

      {/* Table skeleton */}
      <div className="rounded-xl border border-zinc-200 dark:border-zinc-700">
        {/* Header row */}
        <div className="flex gap-4 border-b border-zinc-200 px-4 py-3 dark:border-zinc-700">
          {['w-24', 'w-32', 'w-20', 'w-16', 'w-24'].map((w, i) => (
            <div key={i} className={`h-4 ${w} rounded bg-zinc-200 dark:bg-zinc-700`} />
          ))}
        </div>

        {/* Data rows */}
        {Array.from({ length: rows }).map((_, i) => (
          <div
            key={i}
            className="flex gap-4 border-b border-zinc-100 px-4 py-4 last:border-0 dark:border-zinc-800"
          >
            {['w-24', 'w-40', 'w-16', 'w-20', 'w-28'].map((w, j) => (
              <div key={j} className={`h-4 ${w} rounded bg-zinc-100 dark:bg-zinc-800`} />
            ))}
          </div>
        ))}
      </div>
    </div>
  );
}

// CORRECT: Card skeleton
export function CardSkeleton() {
  return (
    <div className="animate-pulse rounded-xl border border-zinc-200 p-5 dark:border-zinc-700">
      <div className="mb-3 h-4 w-24 rounded bg-zinc-200 dark:bg-zinc-700" />
      <div className="mb-2 h-8 w-32 rounded bg-zinc-200 dark:bg-zinc-700" />
      <div className="h-3 w-48 rounded bg-zinc-100 dark:bg-zinc-800" />
    </div>
  );
}

// CORRECT: Page using skeleton
function OrdersPage() {
  const { data: orders, isLoading, isError } = useOrders();

  if (isLoading) return <TableSkeleton rows={15} />;
  if (isError)   return <ErrorState onRetry={refetch} />;

  return <OrderTable orders={orders} />;
}
```

---

## Lazy Loading — All Routes

```tsx
// router.tsx — ALL pages lazy loaded
import { lazy, Suspense } from 'react';
import { createBrowserRouter, RouterProvider } from 'react-router-dom';

const OrdersIndex  = lazy(() => import('./pages/orders/index'));
const OrderShow    = lazy(() => import('./pages/orders/show'));
const OrderCreate  = lazy(() => import('./pages/orders/create'));
const Dashboard    = lazy(() => import('./pages/dashboard'));

export const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    children: [
      {
        path: 'orders',
        children: [
          {
            index: true,
            element: (
              <Suspense fallback={<TableSkeleton />}>
                <OrdersIndex />
              </Suspense>
            ),
          },
          {
            path: ':id',
            element: (
              <Suspense fallback={<OrderDetailSkeleton />}>
                <OrderShow />
              </Suspense>
            ),
          },
          {
            path: 'create',
            element: (
              <Suspense fallback={<FormSkeleton />}>
                <OrderCreate />
              </Suspense>
            ),
          },
        ],
      },
    ],
  },
]);

// REJECT: Eager import of all pages
import OrdersPage from './pages/orders/index'; // bloats initial bundle
```

---

## Tailwind CSS v4 — Standards

```tsx
// Tailwind v4 uses CSS-first configuration
// tailwind.config.ts is OPTIONAL in v4 — config in CSS:
// @import "tailwindcss";
// @theme { --color-brand: #2563eb; }

// Component pattern with Tailwind
interface BadgeProps {
  status: 'pending' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled';
}

const statusStyles: Record<BadgeProps['status'], string> = {
  pending:   'bg-amber-50 text-amber-700 ring-amber-600/20 dark:bg-amber-950 dark:text-amber-300',
  confirmed: 'bg-blue-50 text-blue-700 ring-blue-600/20 dark:bg-blue-950 dark:text-blue-300',
  shipped:   'bg-purple-50 text-purple-700 ring-purple-600/20',
  delivered: 'bg-green-50 text-green-700 ring-green-600/20 dark:bg-green-950 dark:text-green-300',
  cancelled: 'bg-red-50 text-red-700 ring-red-600/20 dark:bg-red-950 dark:text-red-300',
};

export function OrderStatusBadge({ status }: BadgeProps) {
  return (
    <span
      className={`inline-flex items-center rounded-full px-2 py-1 text-xs font-medium ring-1 ring-inset ${statusStyles[status]}`}
    >
      {status.charAt(0).toUpperCase() + status.slice(1)}
    </span>
  );
}

// REJECT: Inline style for colors
<span style={{ color: 'green' }}>Active</span>

// REJECT: String concatenation for Tailwind classes (breaks purging)
const cls = 'bg-' + color + '-500';  // Tailwind can't detect this — class won't be in bundle
// USE: Full class name in statusStyles map above
```

### Tailwind Component Variants — use `cva`

```tsx
import { cva, type VariantProps } from 'class-variance-authority';

const button = cva(
  'inline-flex items-center justify-center rounded-lg font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        primary:   'bg-blue-600 text-white hover:bg-blue-700 dark:bg-blue-500 dark:hover:bg-blue-600',
        secondary: 'bg-zinc-100 text-zinc-900 hover:bg-zinc-200 dark:bg-zinc-800 dark:text-zinc-100',
        danger:    'bg-red-600 text-white hover:bg-red-700',
        ghost:     'hover:bg-zinc-100 dark:hover:bg-zinc-800',
      },
      size: {
        sm: 'h-8 px-3 text-sm',
        md: 'h-10 px-4 text-sm',
        lg: 'h-11 px-6',
      },
    },
    defaultVariants: {
      variant: 'primary',
      size: 'md',
    },
  }
);

interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof button> {
  loading?: boolean;
}

export function Button({ variant, size, loading, children, ...props }: ButtonProps) {
  return (
    <button className={button({ variant, size })} disabled={loading || props.disabled} {...props}>
      {loading ? <LoadingDots /> : children}
    </button>
  );
}
```

---

## TanStack Query v5 — Data Fetching

```tsx
// hooks/api/useOrders.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

export const orderKeys = {
  all:     ['orders']            as const,
  lists:   () => [...orderKeys.all, 'list']     as const,
  list:    (filters: OrderFilters) => [...orderKeys.lists(), filters] as const,
  details: () => [...orderKeys.all, 'detail']   as const,
  detail:  (id: string) => [...orderKeys.details(), id] as const,
};

export function useOrders(filters: OrderFilters) {
  return useQuery({
    queryKey: orderKeys.list(filters),
    queryFn:  () => ordersApi.list(filters),
    staleTime: 1000 * 60,       // 1 min — don't refetch on every mount
    placeholderData: keepPreviousData, // v5 — no flicker when filters change
  });
}

export function useCreateOrder() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ordersApi.create,
    onSuccess: (newOrder) => {
      // Optimistic: add to cache immediately
      queryClient.setQueryData(orderKeys.detail(newOrder.id), newOrder);
      // Invalidate list so it refetches
      queryClient.invalidateQueries({ queryKey: orderKeys.lists() });
    },
    onError: (error) => {
      toast.error(getErrorMessage(error));
    },
  });
}
```

---

## Common Anti-Patterns

```tsx
// REJECT: Flat page file names
pages/OrdersIndex.tsx
pages/OrderShow.tsx
pages/CreateOrder.tsx
// USE: pages/orders/{index,show,create}.tsx

// REJECT: Spinner for content loading
if (isLoading) return <div className="flex justify-center"><Spinner /></div>;
// USE: <TableSkeleton /> or <CardSkeleton />

// REJECT: Importing entire page (no lazy loading)
import OrdersPage from './pages/orders/index';
// USE: const OrdersPage = lazy(() => import('./pages/orders/index'))

// REJECT: Dynamic Tailwind class construction
className={`text-${color}-500`} // class won't be in bundle

// REJECT: useEffect for data fetching
useEffect(() => {
  fetch('/api/orders').then(r => r.json()).then(setOrders);
}, []);
// USE: useQuery from TanStack Query

// REJECT: Missing Suspense around lazy component
<OrdersPage /> // will throw if OrdersPage is lazy without Suspense wrapper
```

---

## Images — Lazy Loading

```tsx
// CORRECT: Native lazy loading
<img
  src={product.imageUrl}
  alt={product.name}
  loading="lazy"
  decoding="async"
  className="h-48 w-full object-cover"
/>

// CORRECT: Blur placeholder technique
function ProductImage({ src, alt }: { src: string; alt: string }) {
  const [loaded, setLoaded] = useState(false);

  return (
    <div className="relative overflow-hidden rounded-lg bg-zinc-100">
      <img
        src={src}
        alt={alt}
        loading="lazy"
        decoding="async"
        onLoad={() => setLoaded(true)}
        className={`transition-opacity duration-300 ${loaded ? 'opacity-100' : 'opacity-0'}`}
      />
      {!loaded && <div className="absolute inset-0 animate-pulse bg-zinc-200" />}
    </div>
  );
}
```

---

## TypeScript — React 19 Patterns

```tsx
// React 19: ComponentProps helper (preferred over DetailedHTMLProps)
type InputProps = React.ComponentProps<'input'> & {
  label: string;
  error?: string;
};

// No more forwardRef in React 19 — ref is just a prop
function Input({ label, error, ref, ...props }: InputProps) {
  return (
    <div className="flex flex-col gap-1">
      <label className="text-sm font-medium text-zinc-700 dark:text-zinc-300">{label}</label>
      <input
        ref={ref}
        className={`rounded-lg border px-3 py-2 text-sm transition-colors
          ${error
            ? 'border-red-500 focus:ring-red-500'
            : 'border-zinc-300 focus:border-blue-500 focus:ring-blue-500'
          }
          focus:outline-none focus:ring-2`}
        {...props}
      />
      {error && <p className="text-xs text-red-600">{error}</p>}
    </div>
  );
}
```

---

## Deployment Gates

- [ ] All routes lazy-loaded with proper `<Suspense>` fallback
- [ ] Skeleton components match actual page layout
- [ ] No spinners for page-level content (only for button actions)
- [ ] `tsc --noEmit` passes with `strict: true`
- [ ] Bundle analyzed — initial JS chunk < 100KB gzipped
- [ ] All pages in `pages/{domain}/{action}.tsx` — no flat naming
- [ ] Tailwind classes not dynamically constructed with string concat
- [ ] `npm run build` — zero warnings
- [ ] Images use `loading="lazy" decoding="async"`
- [ ] Dark mode works for all skeleton and UI states
