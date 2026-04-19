# Vue 3.5 — SPA Production Skill

## Role

You are a **senior Vue 3 engineer** on Vue 3.5+ + TypeScript 5.5 + Tailwind CSS v4. You use `<script setup>` exclusively, skeleton loading (never spinners for content), lazy-load all routes, and organize pages by domain.

**Version baseline**: Vue 3.5 · TypeScript 5.5 · Tailwind v4 · TanStack Query for Vue v5 · Vue Router 4.x · Pinia 2.x

---

## Vue 3.5 — What Changed (enforce)

```vue
<script setup lang="ts">
// Vue 3.5: useTemplateRef (replaces ref + type cast)
import { useTemplateRef } from 'vue'
const inputRef = useTemplateRef<HTMLInputElement>('input-el')

// Vue 3.5: Reactive props destructuring (no reactivity loss)
const { count = 0, label } = defineProps<{
  count?: number
  label: string
}>()
// count is reactive — NO need for toRefs

// Vue 3.5: defineModel (replaces modelValue + emit)
const value = defineModel<string>({ required: true })

// Vue 3.5: useId() for accessible form elements
import { useId } from 'vue'
const id = useId()
</script>
```

---

## File Structure (domain-first — non-negotiable)

```
src/
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
│       ├── OrderCard.vue
│       ├── OrderStatusBadge.vue
│       ├── OrderFilters.vue
│       └── OrderTable.vue
│
├── composables/
│   ├── api/
│   │   ├── useOrders.ts
│   │   ├── useOrder.ts
│   │   └── useCreateOrder.ts
│   └── useDebounce.ts
│
├── services/
│   └── api/
│       ├── client.ts
│       └── orders.ts
│
├── stores/
│   └── useAuthStore.ts      ← Pinia (global UI state only)
│
├── types/
│   └── index.ts
│
└── router/
    └── index.ts             ← All routes lazy-loaded here
```

**Rule**: `pages/orders/index.vue` — not `pages/OrdersList.vue`, not `pages/orders-index.vue`.

---

## Skeleton Loading — Not Spinners

```vue
<!-- components/ui/skeletons/TableSkeleton.vue -->
<script setup lang="ts">
const { rows = 10 } = defineProps<{ rows?: number }>()
</script>

<template>
  <div class="animate-pulse">
    <!-- Filter bar skeleton -->
    <div class="mb-4 flex gap-3">
      <div class="h-9 w-48 rounded-lg bg-zinc-200 dark:bg-zinc-700" />
      <div class="h-9 w-32 rounded-lg bg-zinc-200 dark:bg-zinc-700" />
    </div>

    <!-- Table skeleton -->
    <div class="rounded-xl border border-zinc-200 dark:border-zinc-700">
      <!-- Header -->
      <div class="flex gap-4 border-b border-zinc-200 px-4 py-3 dark:border-zinc-700">
        <div
          v-for="w in ['w-24', 'w-36', 'w-20', 'w-16', 'w-24']"
          :key="w"
          :class="`h-4 ${w} rounded bg-zinc-200 dark:bg-zinc-700`"
        />
      </div>

      <!-- Rows -->
      <div
        v-for="i in rows"
        :key="i"
        class="flex gap-4 border-b border-zinc-100 px-4 py-4 last:border-0 dark:border-zinc-800"
      >
        <div
          v-for="w in ['w-24', 'w-40', 'w-20', 'w-16', 'w-28']"
          :key="w"
          :class="`h-4 ${w} rounded bg-zinc-100 dark:bg-zinc-800`"
        />
      </div>
    </div>
  </div>
</template>

<!-- pages/orders/index.vue — using skeleton -->
<script setup lang="ts">
const { data: orders, isPending, isError, refetch } = useOrders(filters)
</script>

<template>
  <TableSkeleton v-if="isPending" :rows="15" />
  <ErrorState v-else-if="isError" @retry="refetch" />
  <OrderTable v-else :orders="orders" />
</template>
```

---

## Lazy Loading — All Routes

```typescript
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'

export const router = createRouter({
  history: createWebHistory(),
  routes: [
    {
      path: '/',
      component: () => import('@/layouts/AppLayout.vue'),
      children: [
        {
          path: 'orders',
          children: [
            {
              path: '',
              name: 'orders.index',
              component: () => import('@/pages/orders/index.vue'),
            },
            {
              path: ':id',
              name: 'orders.show',
              component: () => import('@/pages/orders/show.vue'),
            },
            {
              path: 'create',
              name: 'orders.create',
              component: () => import('@/pages/orders/create.vue'),
            },
            {
              path: ':id/edit',
              name: 'orders.edit',
              component: () => import('@/pages/orders/edit.vue'),
            },
          ],
        },
        {
          path: 'products',
          children: [
            { path: '', name: 'products.index', component: () => import('@/pages/products/index.vue') },
            { path: ':id', name: 'products.show',  component: () => import('@/pages/products/show.vue') },
          ],
        },
      ],
    },
    {
      path: '/login',
      name: 'auth.login',
      component: () => import('@/pages/auth/login.vue'),
      meta: { requiresGuest: true },
    },
  ],
})

// REJECT: Eager import
import OrdersPage from '@/pages/orders/index.vue' // bloats initial bundle
```

---

## Tailwind CSS v4 — Component Standards

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
      styles[status]
    ]"
  >
    {{ labels[status] }}
  </span>
</template>
```

```vue
<!-- components/ui/AppButton.vue — with loading state -->
<script setup lang="ts">
type Variant = 'primary' | 'secondary' | 'danger' | 'ghost'
type Size    = 'sm' | 'md' | 'lg'

const {
  variant = 'primary',
  size = 'md',
  loading = false,
} = defineProps<{
  variant?: Variant
  size?: Size
  loading?: boolean
  disabled?: boolean
}>()

const variantStyles: Record<Variant, string> = {
  primary:   'bg-blue-600 text-white hover:bg-blue-700 dark:bg-blue-500',
  secondary: 'bg-zinc-100 text-zinc-900 hover:bg-zinc-200 dark:bg-zinc-800 dark:text-zinc-100',
  danger:    'bg-red-600 text-white hover:bg-red-700',
  ghost:     'hover:bg-zinc-100 dark:hover:bg-zinc-800 text-zinc-700 dark:text-zinc-300',
}

const sizeStyles: Record<Size, string> = {
  sm: 'h-8 px-3 text-sm',
  md: 'h-10 px-4 text-sm',
  lg: 'h-11 px-6 text-base',
}
</script>

<template>
  <button
    :class="[
      'inline-flex items-center justify-center gap-2 rounded-lg font-medium transition-colors',
      'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-blue-500',
      'disabled:pointer-events-none disabled:opacity-50',
      variantStyles[variant],
      sizeStyles[size],
    ]"
    :disabled="loading || disabled"
  >
    <svg v-if="loading" class="h-4 w-4 animate-spin" viewBox="0 0 24 24" fill="none">
      <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4" />
      <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z" />
    </svg>
    <slot />
  </button>
</template>
```

---

## TanStack Query for Vue v5 — Composables

```typescript
// composables/api/useOrders.ts
import { useQuery, useMutation, useQueryClient, keepPreviousData } from '@tanstack/vue-query'
import type { Ref } from 'vue'

export const orderKeys = {
  all:    ['orders']                                     as const,
  lists:  ()              => [...orderKeys.all, 'list'] as const,
  list:   (f: OrderFilters) => [...orderKeys.lists(), f] as const,
  detail: (id: string)    => ['orders', 'detail', id]  as const,
}

export function useOrders(filters: Ref<OrderFilters>) {
  return useQuery({
    queryKey: computed(() => orderKeys.list(filters.value)),
    queryFn:  () => ordersApi.list(filters.value),
    staleTime: 1000 * 60,
    placeholderData: keepPreviousData,
  })
}

export function useCreateOrder() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: ordersApi.create,
    onSuccess: (order) => {
      queryClient.invalidateQueries({ queryKey: orderKeys.lists() })
      queryClient.setQueryData(orderKeys.detail(order.id), order)
    },
  })
}
```

---

## Common Anti-Patterns

```vue
<!-- REJECT: Options API -->
<script>
export default {
  data() { return { orders: [] } }
}
</script>

<!-- REJECT: Spinner for content -->
<template>
  <div v-if="loading" class="flex justify-center py-10">
    <Spinner />
  </div>
</template>

<!-- REJECT: Dynamic Tailwind class construction -->
<div :class="`bg-${status}-500`">  <!-- class won't be in bundle -->

<!-- REJECT: Non-lazy route -->
import OrdersPage from '@/pages/orders/index.vue'

<!-- REJECT: Flat page names -->
<!-- pages/OrdersList.vue, pages/orders-index.vue -->

<!-- REJECT: Missing lang="ts" -->
<script setup>
```

---

## Images — Lazy Loading

```vue
<template>
  <div class="relative overflow-hidden rounded-lg bg-zinc-100 dark:bg-zinc-800">
    <img
      :src="src"
      :alt="alt"
      loading="lazy"
      decoding="async"
      :class="['w-full object-cover transition-opacity duration-300', loaded ? 'opacity-100' : 'opacity-0']"
      @load="loaded = true"
    />
    <div
      v-if="!loaded"
      class="absolute inset-0 animate-pulse bg-zinc-200 dark:bg-zinc-700"
    />
  </div>
</template>

<script setup lang="ts">
const { src, alt } = defineProps<{ src: string; alt: string }>()
const loaded = ref(false)
</script>
```

---

## Deployment Gates

- [ ] `vue-tsc --noEmit` passes — zero type errors
- [ ] All routes use `() => import(...)` — no eager imports
- [ ] Skeleton components exist for all loading states — no content spinners
- [ ] All pages in `pages/{domain}/{action}.vue`
- [ ] No Options API (`grep -r "export default {" src/pages` is empty)
- [ ] All Tailwind classes are static strings — no dynamic concatenation
- [ ] Dark mode works across all skeleton/UI states
- [ ] Images use `loading="lazy" decoding="async"`
- [ ] Bundle analyzed — initial chunk < 100KB gzipped
