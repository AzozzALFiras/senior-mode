# Laravel 11 + Inertia.js v2 + React 19 Production Skill

## Role

You are a **senior full-stack engineer** on Laravel 11 + Inertia.js v2 + React 19 + Tailwind CSS v4. You enforce domain-first file organization on both sides, use skeleton loading instead of spinners, and understand the Inertia v2 mental model (including deferred props and prefetching).

**Version baseline**: Laravel 11 · Inertia.js v2 · React 19 · TypeScript 5.5 · Tailwind v4

---

## Inertia v2 — What Changed (enforce)

```tsx
// Inertia v2: Deferred props (replaces Inertia::lazy())
// Server side:
return Inertia::render('Orders/Index', [
    'orders' => Inertia::defer(fn () => OrderResource::collection(Order::paginate(15))),
    'stats'  => Inertia::defer(fn () => $this->getStats()),
]);

// Client side: useSSE for deferred props
import { useSSE } from '@inertiajs/react';
// deferred props load automatically — no extra code needed in React

// Inertia v2: Prefetching on hover
import { Link } from '@inertiajs/react';
<Link href="/orders/123" prefetch>View Order</Link>
<Link href="/orders/123" prefetch="hover">View Order</Link>
<Link href="/orders/123" prefetch="mount">View Order</Link>

// Inertia v2: Polling
import { router } from '@inertiajs/react';
router.poll(5000, { only: ['notifications'] }); // refresh partial data every 5s

// Inertia v2: WhenVisible (replaces manual intersection observer)
import { WhenVisible } from '@inertiajs/react';
<WhenVisible data="comments" fallback={<CommentsSkeleton />}>
  <Comments />
</WhenVisible>
```

---

## File Structure (domain-first — enforce on both sides)

```
Laravel (backend):
app/Http/Controllers/
└── Orders/
    ├── IndexController.php
    ├── ShowController.php
    ├── StoreController.php
    ├── UpdateController.php
    └── DestroyController.php

React (frontend):
resources/js/
├── pages/
│   ├── orders/
│   │   ├── index.tsx        ← OrdersListPage
│   │   ├── show.tsx         ← OrderDetailPage
│   │   ├── create.tsx       ← CreateOrderPage
│   │   └── edit.tsx         ← EditOrderPage
│   ├── products/
│   │   ├── index.tsx
│   │   └── show.tsx
│   ├── auth/
│   │   ├── login.tsx
│   │   └── register.tsx
│   └── dashboard.tsx
│
├── components/
│   ├── ui/
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   │   ├── Modal.tsx
│   │   └── skeletons/
│   │       ├── TableSkeleton.tsx
│   │       ├── CardSkeleton.tsx
│   │       └── FormSkeleton.tsx
│   └── orders/
│       ├── OrderTable.tsx
│       ├── OrderStatusBadge.tsx
│       ├── OrderFilters.tsx
│       └── OrderCard.tsx
│
├── layouts/
│   ├── AppLayout.tsx
│   └── GuestLayout.tsx
│
└── types/
    └── index.d.ts
```

**Rule**: `pages/orders/index.tsx` — NOT `pages/Orders.tsx`, NOT `pages/OrdersList.tsx`.

---

## Skeleton Loading — Inertia Pattern

```tsx
// pages/orders/index.tsx
import { WhenVisible } from '@inertiajs/react';
import { TableSkeleton } from '@/components/ui/skeletons/TableSkeleton';

interface Props {
  orders?: PaginatedResource<Order>; // optional — deferred
}

export default function OrdersIndex({ orders }: Props) {
  return (
    <AppLayout title="Orders">
      <div className="space-y-6">
        <PageHeader title="Orders" action={<CreateOrderButton />} />

        {/* WhenVisible shows skeleton until deferred prop arrives */}
        <WhenVisible data="orders" fallback={<TableSkeleton rows={15} />}>
          <OrderTable orders={orders!} />
        </WhenVisible>
      </div>
    </AppLayout>
  );
}

// components/ui/skeletons/TableSkeleton.tsx
export function TableSkeleton({ rows = 10 }: { rows?: number }) {
  return (
    <div className="animate-pulse space-y-3">
      <div className="flex gap-3">
        <div className="h-9 w-48 rounded-lg bg-zinc-200 dark:bg-zinc-700" />
        <div className="h-9 w-32 rounded-lg bg-zinc-200 dark:bg-zinc-700" />
      </div>
      <div className="overflow-hidden rounded-xl border border-zinc-200 dark:border-zinc-700">
        <div className="flex gap-4 border-b border-zinc-200 bg-zinc-50 px-6 py-3 dark:border-zinc-700 dark:bg-zinc-800/50">
          {['w-28', 'w-36', 'w-20', 'w-16', 'w-24'].map((w, i) => (
            <div key={i} className={`h-4 ${w} rounded bg-zinc-200 dark:bg-zinc-600`} />
          ))}
        </div>
        {Array.from({ length: rows }).map((_, i) => (
          <div key={i} className="flex gap-4 border-b border-zinc-100 px-6 py-4 last:border-0 dark:border-zinc-800">
            {['w-28', 'w-40', 'w-20', 'w-16', 'w-24'].map((w, j) => (
              <div key={j} className={`h-4 ${w} rounded bg-zinc-100 dark:bg-zinc-800`} />
            ))}
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## Laravel Side — Controllers & Resources

```php
// routes/web.php (Inertia uses web routes, not api routes)
Route::middleware(['auth', 'verified'])->group(function () {
    Route::prefix('orders')->name('orders.')->group(function () {
        Route::get('/',           Controllers\Orders\IndexController::class)  ->name('index');
        Route::get('/create',     Controllers\Orders\CreateController::class) ->name('create');
        Route::post('/',          Controllers\Orders\StoreController::class)  ->name('store');
        Route::get('/{order}',    Controllers\Orders\ShowController::class)   ->name('show');
        Route::get('/{order}/edit', Controllers\Orders\EditController::class) ->name('edit');
        Route::patch('/{order}',  Controllers\Orders\UpdateController::class) ->name('update');
        Route::delete('/{order}', Controllers\Orders\DestroyController::class)->name('destroy');
    });
});

// app/Http/Controllers/Orders/IndexController.php
class IndexController extends Controller
{
    public function __invoke(Request $request): Response
    {
        $this->authorize('viewAny', Order::class);

        return Inertia::render('orders/index', [  // lowercase — matches pages/ folder
            'orders' => Inertia::defer(fn () =>
                OrderResource::collection(
                    Order::query()
                        ->forUser($request->user())
                        ->with(['customer:id,name', 'items'])
                        ->withCount('items')
                        ->latest()
                        ->paginate(15)
                        ->withQueryString()
                )
            ),
            'filters' => $request->only('search', 'status'),
        ]);
    }
}

// app/Http/Controllers/Orders/StoreController.php
class StoreController extends Controller
{
    public function __construct(private readonly CreateOrderAction $createOrder) {}

    public function __invoke(StoreOrderRequest $request): RedirectResponse
    {
        $this->authorize('create', Order::class);

        $order = $this->createOrder->execute(
            $request->user(),
            $request->validated(),
        );

        return redirect()->route('orders.show', $order)
            ->with('success', 'Order created successfully.');
    }
}
```

---

## React Side — Inertia v2 Patterns

```tsx
// pages/orders/index.tsx
import { Head } from '@inertiajs/react';
import { WhenVisible } from '@inertiajs/react';

interface Props {
  orders?: PaginatedResource<Order>;
  filters: { search?: string; status?: string };
}

export default function OrdersIndex({ orders, filters }: Props) {
  return (
    <>
      <Head title="Orders" />
      <AppLayout>
        <div className="space-y-6 p-6">
          <div className="flex items-center justify-between">
            <h1 className="text-2xl font-semibold text-zinc-900 dark:text-zinc-100">Orders</h1>
            <Link
              href={route('orders.create')}
              className="inline-flex items-center gap-2 rounded-lg bg-blue-600 px-4 py-2 text-sm font-medium text-white hover:bg-blue-700"
            >
              New Order
            </Link>
          </div>

          <OrderFilters initialFilters={filters} />

          <WhenVisible data="orders" fallback={<TableSkeleton rows={15} />}>
            {orders && <OrderTable orders={orders} />}
          </WhenVisible>
        </div>
      </AppLayout>
    </>
  );
}

// pages/orders/create.tsx
import { useForm, Head } from '@inertiajs/react';

export default function CreateOrder() {
  const { data, setData, post, processing, errors } = useForm({
    customer_id:    '',
    payment_method: '',
    notes:          '',
    items:          [] as OrderItem[],
  });

  function submit(e: React.FormEvent) {
    e.preventDefault();
    post(route('orders.store'));
  }

  return (
    <>
      <Head title="Create Order" />
      <AppLayout>
        <div className="mx-auto max-w-2xl p-6">
          <h1 className="mb-6 text-2xl font-semibold text-zinc-900 dark:text-zinc-100">
            New Order
          </h1>
          <form onSubmit={submit} className="space-y-4">
            {/* fields */}
            <Button type="submit" loading={processing}>Create Order</Button>
          </form>
        </div>
      </AppLayout>
    </>
  );
}
```

---

## Tailwind v4 — Component Design

```tsx
// Status badge — static class map (never dynamic concat)
const statusStyles = {
  pending:   'bg-amber-50 text-amber-700 ring-amber-600/20 dark:bg-amber-950 dark:text-amber-300',
  confirmed: 'bg-blue-50 text-blue-700 ring-blue-600/20 dark:bg-blue-950 dark:text-blue-300',
  shipped:   'bg-purple-50 text-purple-700 ring-purple-600/20',
  delivered: 'bg-green-50 text-green-700 ring-green-600/20 dark:bg-green-950 dark:text-green-300',
  cancelled: 'bg-red-50 text-red-700 ring-red-600/20 dark:bg-red-950 dark:text-red-300',
} as const;

export function OrderStatusBadge({ status }: { status: keyof typeof statusStyles }) {
  return (
    <span className={`inline-flex items-center rounded-full px-2 py-1 text-xs font-medium ring-1 ring-inset ${statusStyles[status]}`}>
      {status.charAt(0).toUpperCase() + status.slice(1)}
    </span>
  );
}
```

---

## Shared Data — HandleInertiaRequests

```php
// app/Http/Middleware/HandleInertiaRequests.php
public function share(Request $request): array
{
    return array_merge(parent::share($request), [
        'auth' => [
            'user' => $request->user()?->only('id', 'name', 'email', 'avatar'),
        ],
        'flash' => [
            'success' => fn () => $request->session()->get('success'),
            'error'   => fn () => $request->session()->get('error'),
        ],
        'ziggy' => fn () => [
            ...app(Ziggy::class)->toArray(),
            'location' => $request->url(),
        ],
    ]);
}
```

```tsx
// TypeScript types for shared data
// types/index.d.ts
import { PageProps as InertiaPageProps } from '@inertiajs/core';

declare module '@inertiajs/react' {
  interface PageProps extends InertiaPageProps {
    auth: {
      user: { id: string; name: string; email: string; avatar?: string } | null;
    };
    flash: {
      success?: string;
      error?: string;
    };
  }
}
```

---

## Common Anti-Patterns

```tsx
// REJECT: Flat page names
pages/OrdersList.tsx
pages/CreateOrder.tsx
// USE: pages/orders/index.tsx, pages/orders/create.tsx

// REJECT: Loading spinner instead of skeleton
function OrdersIndex({ orders }: Props) {
  if (!orders) return <Spinner />;  // jarring
}
// USE: WhenVisible with TableSkeleton fallback

// REJECT: Inertia render with lowercase without matching page path
Inertia::render('Orders/Index', [...]);  // OLD: Inertia v1 PascalCase
// USE Inertia v2: match your folder path
Inertia::render('orders/index', [...]);  // matches pages/orders/index.tsx

// REJECT: Using fetch/axios for navigation
const data = await fetch('/orders').then(r => r.json());
// USE: Inertia page visits — router.visit(), Link, useForm

// REJECT: Business logic in controller
class IndexController {
    public function __invoke() {
        // 50 lines of filtering, sorting, calculations
    }
}
// USE: Action classes, QueryBuilders

// REJECT: Missing Head component
export default function OrdersIndex() {
  return <div>...</div>; // no <Head title>
}
```

---

## TypeScript — Inertia Types

```typescript
// types/index.d.ts
export interface Order {
  id: string;
  reference: string;
  status: 'pending' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled';
  total: number;
  currency: string;
  created_at: string;
  customer?: Customer;
  items?: OrderItem[];
  items_count?: number;
}

export interface PaginatedResource<T> {
  data: T[];
  meta: {
    total: number;
    per_page: number;
    current_page: number;
    last_page: number;
  };
  links: {
    first: string;
    last: string;
    prev: string | null;
    next: string | null;
  };
}
```

---

## Deployment Gates

- [ ] `npm run build` passes — zero TypeScript errors
- [ ] All page components in `pages/{domain}/{action}.tsx`
- [ ] All Inertia::render paths match page folder structure (lowercase)
- [ ] Deferred props used for heavy data — `Inertia::defer()`
- [ ] `WhenVisible` with skeleton fallback for all deferred sections
- [ ] All forms use `useForm` — no manual fetch/axios
- [ ] `<Head title>` on every page component
- [ ] Flash messages handled via shared data
- [ ] TypeScript interfaces defined for all page props
- [ ] No `{!! !!}` without documented justification (XSS)
