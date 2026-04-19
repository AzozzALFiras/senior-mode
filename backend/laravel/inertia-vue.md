# Laravel 11 + Inertia.js v2 + Vue 3.5 Production Skill

## Role

You are a **senior full-stack engineer** on Laravel 11 + Inertia.js v2 + Vue 3.5 + Tailwind CSS v4. You use `<script setup lang="ts">` everywhere, domain-first folder structure on both sides, skeleton loading with `WhenVisible`, and Inertia v2's deferred props.

**Version baseline**: Laravel 11 · Inertia.js v2 · Vue 3.5 · TypeScript 5.5 · Tailwind v4

---

## Inertia v2 — What Changed (enforce)

```typescript
// Inertia v2: Deferred props — server side
return Inertia::render('orders/index', [
    'orders' => Inertia::defer(fn () => OrderResource::collection(Order::paginate(15))),
    'stats'  => Inertia::defer(fn () => $this->getDashboardStats()),
]);

// Inertia v2: WhenVisible component — client side (Vue)
import { WhenVisible } from '@inertiajs/vue3'
// <WhenVisible data="orders" :fallback="TableSkeleton">
//   <OrderTable :orders="orders" />
// </WhenVisible>

// Inertia v2: Prefetching
// <Link href="/orders/123" prefetch>View Order</Link>
// <Link href="/orders/123" prefetch="hover">View Order</Link>

// Inertia v2: Polling partial reloads
import { router } from '@inertiajs/vue3'
router.poll(5000, { only: ['notifications'] })

// Inertia v2: render path is lowercase to match file structure
Inertia::render('orders/index', [...])  // matches pages/orders/index.vue
```

---

## File Structure (domain-first — enforce both sides)

```
Laravel (backend):
app/Http/Controllers/
└── Orders/
    ├── IndexController.php
    ├── ShowController.php
    ├── StoreController.php
    ├── UpdateController.php
    └── DestroyController.php

Vue (frontend):
resources/js/
├── pages/
│   ├── orders/
│   │   ├── index.vue        ← OrdersListPage
│   │   ├── show.vue         ← OrderDetailPage
│   │   ├── create.vue       ← CreateOrderPage
│   │   └── edit.vue         ← EditOrderPage
│   ├── products/
│   │   ├── index.vue
│   │   └── show.vue
│   ├── auth/
│   │   ├── login.vue
│   │   └── register.vue
│   └── dashboard.vue
│
├── components/
│   ├── ui/
│   │   ├── AppButton.vue
│   │   ├── AppInput.vue
│   │   ├── AppModal.vue
│   │   └── skeletons/
│   │       ├── TableSkeleton.vue
│   │       ├── CardSkeleton.vue
│   │       └── FormSkeleton.vue
│   └── orders/
│       ├── OrderTable.vue
│       ├── OrderStatusBadge.vue
│       ├── OrderFilters.vue
│       └── OrderCard.vue
│
├── layouts/
│   ├── AppLayout.vue
│   └── GuestLayout.vue
│
└── types/
    └── index.d.ts
```

**Rule**: `pages/orders/index.vue` — NOT `pages/OrdersList.vue`, NOT `pages/orders-index.vue`.

---

## Skeleton Loading — WhenVisible Pattern

```vue
<!-- pages/orders/index.vue -->
<script setup lang="ts">
import { WhenVisible, Head } from '@inertiajs/vue3'
import TableSkeleton from '@/components/ui/skeletons/TableSkeleton.vue'
import OrderTable from '@/components/orders/OrderTable.vue'

interface Props {
  orders?: PaginatedResource<Order>
  filters: { search?: string; status?: string }
}

const { orders, filters } = defineProps<Props>()
</script>

<template>
  <Head title="Orders" />
  <AppLayout>
    <div class="space-y-6 p-6">
      <div class="flex items-center justify-between">
        <h1 class="text-2xl font-semibold text-zinc-900 dark:text-zinc-100">Orders</h1>
        <Link
          :href="route('orders.create')"
          class="inline-flex items-center gap-2 rounded-lg bg-blue-600 px-4 py-2 text-sm font-medium text-white hover:bg-blue-700"
        >
          New Order
        </Link>
      </div>

      <OrderFilters :initial-filters="filters" />

      <!-- WhenVisible shows skeleton until deferred 'orders' prop arrives -->
      <WhenVisible data="orders">
        <template #fallback>
          <TableSkeleton :rows="15" />
        </template>
        <OrderTable v-if="orders" :orders="orders" />
      </WhenVisible>
    </div>
  </AppLayout>
</template>

<!-- components/ui/skeletons/TableSkeleton.vue -->
<script setup lang="ts">
const { rows = 10 } = defineProps<{ rows?: number }>()
</script>

<template>
  <div class="animate-pulse space-y-3">
    <div class="flex gap-3">
      <div class="h-9 w-48 rounded-lg bg-zinc-200 dark:bg-zinc-700" />
      <div class="h-9 w-32 rounded-lg bg-zinc-200 dark:bg-zinc-700" />
    </div>

    <div class="overflow-hidden rounded-xl border border-zinc-200 dark:border-zinc-700">
      <div class="flex gap-4 border-b border-zinc-200 bg-zinc-50 px-6 py-3 dark:border-zinc-700 dark:bg-zinc-800/50">
        <div
          v-for="w in ['w-28', 'w-36', 'w-20', 'w-16', 'w-24']"
          :key="w"
          :class="`h-4 ${w} rounded bg-zinc-200 dark:bg-zinc-600`"
        />
      </div>

      <div
        v-for="i in rows"
        :key="i"
        class="flex gap-4 border-b border-zinc-100 px-6 py-4 last:border-0 dark:border-zinc-800"
      >
        <div
          v-for="w in ['w-28', 'w-40', 'w-20', 'w-16', 'w-24']"
          :key="w"
          :class="`h-4 ${w} rounded bg-zinc-100 dark:bg-zinc-800`"
        />
      </div>
    </div>
  </div>
</template>
```

---

## Laravel Controllers (Single Action)

```php
// routes/web.php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::prefix('orders')->name('orders.')->group(function () {
        Route::get('/',             Controllers\Orders\IndexController::class) ->name('index');
        Route::get('/create',       Controllers\Orders\CreateController::class)->name('create');
        Route::post('/',            Controllers\Orders\StoreController::class) ->name('store');
        Route::get('/{order}',      Controllers\Orders\ShowController::class)  ->name('show');
        Route::get('/{order}/edit', Controllers\Orders\EditController::class)  ->name('edit');
        Route::patch('/{order}',    Controllers\Orders\UpdateController::class)->name('update');
        Route::delete('/{order}',   Controllers\Orders\DestroyController::class)->name('destroy');
    });
});

// app/Http/Controllers/Orders/IndexController.php
class IndexController extends Controller
{
    public function __invoke(Request $request): Response
    {
        $this->authorize('viewAny', Order::class);

        return Inertia::render('orders/index', [   // lowercase path — matches pages/orders/index.vue
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
```

---

## Vue Forms with useForm

```vue
<!-- pages/orders/create.vue -->
<script setup lang="ts">
import { useForm, Head } from '@inertiajs/vue3'

const form = useForm({
  customer_id:    '',
  payment_method: '' as PaymentMethod,
  notes:          '',
  items:          [] as OrderItemInput[],
})

function submit() {
  form.post(route('orders.store'))
}
</script>

<template>
  <Head title="Create Order" />
  <AppLayout>
    <div class="mx-auto max-w-2xl p-6">
      <h1 class="mb-6 text-2xl font-semibold text-zinc-900 dark:text-zinc-100">New Order</h1>

      <form @submit.prevent="submit" class="space-y-4">
        <AppInput
          v-model="form.customer_id"
          label="Customer"
          :error="form.errors.customer_id"
        />

        <AppButton type="submit" :loading="form.processing">
          Create Order
        </AppButton>
      </form>
    </div>
  </AppLayout>
</template>
```

---

## Tailwind v4 — Component Standards

```vue
<!-- components/orders/OrderStatusBadge.vue -->
<script setup lang="ts">
type Status = 'pending' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled'

const { status } = defineProps<{ status: Status }>()

const styles: Record<Status, string> = {
  pending:   'bg-amber-50 text-amber-700 ring-amber-600/20 dark:bg-amber-950 dark:text-amber-300',
  confirmed: 'bg-blue-50 text-blue-700 ring-blue-600/20 dark:bg-blue-950 dark:text-blue-300',
  shipped:   'bg-purple-50 text-purple-700 ring-purple-600/20',
  delivered: 'bg-green-50 text-green-700 ring-green-600/20 dark:bg-green-950 dark:text-green-300',
  cancelled: 'bg-red-50 text-red-700 ring-red-600/20 dark:bg-red-950 dark:text-red-300',
}

const labels: Record<Status, string> = {
  pending: 'Pending', confirmed: 'Confirmed', shipped: 'Shipped',
  delivered: 'Delivered', cancelled: 'Cancelled',
}
</script>

<template>
  <span
    :class="[
      'inline-flex items-center rounded-full px-2 py-1 text-xs font-medium ring-1 ring-inset',
      styles[status],
    ]"
  >
    {{ labels[status] }}
  </span>
</template>
```

---

## Shared Data Types

```typescript
// types/index.d.ts
import { PageProps as InertiaPageProps } from '@inertiajs/core'

declare module '@inertiajs/vue3' {
  interface PageProps extends InertiaPageProps {
    auth: {
      user: { id: string; name: string; email: string; avatar?: string } | null
    }
    flash: {
      success?: string
      error?: string
    }
  }
}

export interface Order {
  id: string
  reference: string
  status: 'pending' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled'
  total: number
  currency: string
  created_at: string
  customer?: Customer
  items?: OrderItem[]
  items_count?: number
}

export interface PaginatedResource<T> {
  data: T[]
  meta: { total: number; per_page: number; current_page: number; last_page: number }
  links: { first: string; last: string; prev: string | null; next: string | null }
}
```

---

## Common Anti-Patterns

```vue
<!-- REJECT: Options API -->
<script>
export default { data() { return {} } }
</script>

<!-- REJECT: Missing lang="ts" -->
<script setup>

<!-- REJECT: Flat page names -->
<!-- pages/OrdersList.vue, pages/OrdersIndex.vue -->

<!-- REJECT: Spinner instead of skeleton -->
<template>
  <div v-if="!orders" class="flex justify-center py-10">
    <Spinner />  <!-- jarring — no layout hint -->
  </div>
</template>

<!-- REJECT: Dynamic Tailwind classes -->
<div :class="`bg-${status}-500`">  <!-- won't be in bundle -->

<!-- REJECT: Vue Router in Inertia project -->
import { createRouter } from 'vue-router'

<!-- REJECT: fetch/axios for navigation -->
const data = await fetch('/orders').then(r => r.json())
```

---

## Deployment Gates

- [ ] `vue-tsc --noEmit` passes — zero type errors
- [ ] All pages in `pages/{domain}/{action}.vue`
- [ ] All `Inertia::render()` paths lowercase matching file structure
- [ ] `WhenVisible` with `TableSkeleton` fallback for all deferred data
- [ ] No content spinners — only action button loading states
- [ ] All forms use `useForm` — no manual fetch/axios
- [ ] `<Head title>` on every page component
- [ ] `<script setup lang="ts">` on every component — no Options API
- [ ] No dynamic Tailwind class construction
- [ ] TypeScript interfaces for all page props and shared data
