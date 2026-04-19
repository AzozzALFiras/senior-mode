# Next.js 15 — App Router Production Skill

## Role

You are a **senior Next.js engineer** on Next.js 15 + React 19 + TypeScript 5.5 + Tailwind CSS v4. You use Server Components by default, skeleton loading via `loading.tsx`, lazy-load heavy Client Components, and organize pages by domain.

**Version baseline**: Next.js 15 · React 19 · TypeScript 5.5 · Tailwind v4 · TanStack Query v5

---

## Next.js 15 — What Changed (enforce)

```tsx
// Next.js 15: Dynamic APIs are now async (breaking change)
// params and searchParams are Promises — must await them

// REJECT (Next.js 14 pattern — broken in 15):
export default function OrderPage({ params }: { params: { id: string } }) {
  const { id } = params; // sync access — error in Next.js 15
}

// CORRECT (Next.js 15):
export default async function OrderPage({
  params,
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params;
  const order = await getOrder(id);
  return <OrderDetail order={order} />;
}

// searchParams also async:
export default async function OrdersPage({
  searchParams,
}: {
  searchParams: Promise<{ page?: string; status?: string }>
}) {
  const { page = '1', status } = await searchParams;
  // ...
}

// Next.js 15: fetch() no longer cached by default
// Explicit: cache: 'force-cache' for caching, 'no-store' for always fresh
const data = await fetch('/api/orders', { cache: 'no-store' });
const product = await fetch(`/api/products/${id}`, { next: { revalidate: 3600 } });

// Next.js 15: Turbopack is stable and default in dev
// next dev  → uses Turbopack
// next dev --turbopack  → explicit (same)
```

---

## File Structure (domain-first — non-negotiable)

```
app/
├── (auth)/                        ← Route group — no URL segment
│   ├── login/
│   │   └── page.tsx
│   ├── register/
│   │   └── page.tsx
│   └── layout.tsx                 ← Guest layout
│
├── (dashboard)/                   ← Route group
│   ├── layout.tsx                 ← Dashboard layout (sidebar, navbar)
│   │
│   ├── orders/                    ← Domain folder
│   │   ├── page.tsx               ← Orders list (Server Component)
│   │   ├── loading.tsx            ← Skeleton — auto shown by Next.js
│   │   ├── error.tsx              ← Error boundary
│   │   ├── actions.ts             ← Server Actions ('use server')
│   │   │
│   │   └── [id]/
│   │       ├── page.tsx           ← Order detail
│   │       ├── loading.tsx
│   │       └── edit/
│   │           ├── page.tsx
│   │           └── loading.tsx
│   │
│   ├── products/
│   │   ├── page.tsx
│   │   ├── loading.tsx
│   │   └── [id]/
│   │       └── page.tsx
│   │
│   └── dashboard/
│       ├── page.tsx
│       └── loading.tsx
│
├── api/
│   └── webhooks/
│       └── stripe/
│           └── route.ts
│
├── layout.tsx                     ← Root layout
├── not-found.tsx
└── global-error.tsx

components/
├── ui/                            ← Generic, no 'use client' unless needed
│   ├── Button.tsx
│   ├── Input.tsx
│   └── skeletons/
│       ├── TableSkeleton.tsx
│       ├── CardSkeleton.tsx
│       └── StatCardSkeleton.tsx
└── orders/
    ├── OrderTable.tsx             ← Server Component
    ├── OrderStatusBadge.tsx       ← Server Component (no interactivity)
    ├── CancelOrderButton.tsx      ← 'use client'
    └── OrderFilters.tsx           ← 'use client' (interactive)
```

**Rule**: `app/(dashboard)/orders/page.tsx` — not `app/pages/OrdersList.tsx`.

---

## loading.tsx — Skeleton Loading Standard

```tsx
// app/(dashboard)/orders/loading.tsx
// Next.js shows this automatically during page load — no code needed in page.tsx
export default function OrdersLoading() {
  return <OrdersTableSkeleton />;
}

// components/ui/skeletons/TableSkeleton.tsx (Server Component — no 'use client')
export function OrdersTableSkeleton() {
  return (
    <div className="animate-pulse space-y-4">
      {/* Page header skeleton */}
      <div className="flex items-center justify-between">
        <div className="h-8 w-32 rounded-lg bg-zinc-200 dark:bg-zinc-700" />
        <div className="h-10 w-28 rounded-lg bg-zinc-200 dark:bg-zinc-700" />
      </div>

      {/* Filter bar skeleton */}
      <div className="flex gap-3">
        <div className="h-9 w-64 rounded-lg bg-zinc-200 dark:bg-zinc-700" />
        <div className="h-9 w-32 rounded-lg bg-zinc-200 dark:bg-zinc-700" />
        <div className="h-9 w-24 rounded-lg bg-zinc-200 dark:bg-zinc-700" />
      </div>

      {/* Table skeleton */}
      <div className="overflow-hidden rounded-xl border border-zinc-200 dark:border-zinc-700">
        {/* Header */}
        <div className="flex gap-4 border-b border-zinc-200 bg-zinc-50 px-6 py-3 dark:border-zinc-700 dark:bg-zinc-800/50">
          {['w-28', 'w-36', 'w-20', 'w-24', 'w-20'].map((w, i) => (
            <div key={i} className={`h-4 ${w} rounded bg-zinc-200 dark:bg-zinc-600`} />
          ))}
        </div>

        {/* Rows */}
        {Array.from({ length: 10 }).map((_, i) => (
          <div
            key={i}
            className="flex gap-4 border-b border-zinc-100 px-6 py-4 last:border-0 dark:border-zinc-800"
          >
            {['w-28', 'w-40', 'w-16', 'w-20', 'w-24'].map((w, j) => (
              <div key={j} className={`h-4 ${w} rounded bg-zinc-100 dark:bg-zinc-800`} />
            ))}
          </div>
        ))}
      </div>
    </div>
  );
}

// app/(dashboard)/orders/[id]/loading.tsx
export default function OrderDetailLoading() {
  return <OrderDetailSkeleton />;
}
```

---

## Server Components — Data Fetching

```tsx
// app/(dashboard)/orders/page.tsx
import { db } from '@/lib/db';
import { OrderTable } from '@/components/orders/OrderTable';
import { auth } from '@/lib/auth';
import { redirect } from 'next/navigation';

export const metadata = {
  title: 'Orders | MyApp',
};

interface Props {
  searchParams: Promise<{ page?: string; status?: string; search?: string }>;
}

export default async function OrdersPage({ searchParams }: Props) {
  const session = await auth();
  if (!session) redirect('/login');

  const { page = '1', status, search } = await searchParams;
  const pageNum = Math.max(1, parseInt(page));

  // Direct DB query in Server Component — no API round trip needed
  const [orders, total] = await Promise.all([
    db.order.findMany({
      where: {
        userId: session.user.id,
        ...(status && { status }),
        ...(search && { reference: { contains: search, mode: 'insensitive' } }),
      },
      include: {
        customer: { select: { id: true, name: true, email: true } },
        _count: { select: { items: true } },
      },
      orderBy: { createdAt: 'desc' },
      take: 15,
      skip: (pageNum - 1) * 15,
    }),
    db.order.count({ where: { userId: session.user.id } }),
  ]);

  return (
    <div className="space-y-6">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-semibold text-zinc-900 dark:text-zinc-100">Orders</h1>
        <CreateOrderButton /> {/* 'use client' */}
      </div>

      <Suspense fallback={<OrderFiltersLoading />}>
        <OrderFilters />   {/* 'use client' — interactive */}
      </Suspense>

      <OrderTable orders={orders} total={total} page={pageNum} />
    </div>
  );
}
```

---

## Server Actions — Next.js 15

```tsx
// app/(dashboard)/orders/actions.ts
'use server';

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';
import { z } from 'zod';
import { auth } from '@/lib/auth';
import { db } from '@/lib/db';

const CreateOrderSchema = z.object({
  customerId:    z.string().uuid(),
  paymentMethod: z.enum(['stripe', 'bank_transfer']),
  notes:         z.string().max(1000).optional(),
  items: z.array(z.object({
    productId: z.string().uuid(),
    quantity:  z.number().int().positive(),
    unitPrice: z.number().positive(),
  })).min(1),
});

export async function createOrder(prevState: unknown, formData: FormData) {
  // 1. Auth check — ALWAYS
  const session = await auth();
  if (!session) return { error: 'Unauthorized' };

  // 2. Validate
  const raw = {
    customerId:    formData.get('customerId'),
    paymentMethod: formData.get('paymentMethod'),
    notes:         formData.get('notes') || undefined,
    items:         JSON.parse(formData.get('items') as string),
  };

  const result = CreateOrderSchema.safeParse(raw);
  if (!result.success) {
    return { errors: result.error.flatten().fieldErrors };
  }

  // 3. Authorization — verify customer belongs to user
  const customer = await db.customer.findFirst({
    where: { id: result.data.customerId, userId: session.user.id },
  });
  if (!customer) return { error: 'Customer not found' };

  // 4. Execute
  const order = await db.order.create({
    data: {
      userId:        session.user.id,
      customerId:    result.data.customerId,
      paymentMethod: result.data.paymentMethod,
      notes:         result.data.notes,
      status:        'pending',
      total: result.data.items.reduce((sum, i) => sum + i.quantity * i.unitPrice, 0),
      items: { createMany: { data: result.data.items } },
    },
  });

  revalidatePath('/orders');
  redirect(`/orders/${order.id}`);
}

// Client Component using the action
'use client';
import { useActionState } from 'react';
import { createOrder } from '../actions';

export function CreateOrderForm() {
  const [state, action, isPending] = useActionState(createOrder, null);

  return (
    <form action={action} className="space-y-4">
      {state?.error && (
        <p className="rounded-lg bg-red-50 p-3 text-sm text-red-700">{state.error}</p>
      )}
      {/* form fields */}
      <Button type="submit" loading={isPending}>
        Create Order
      </Button>
    </form>
  );
}
```

---

## Tailwind CSS v4 — Standards

```tsx
// Tailwind v4: CSS-first config
// In globals.css:
// @import "tailwindcss";
// @theme {
//   --color-brand: oklch(0.55 0.2 250);
//   --font-sans: 'Inter', sans-serif;
// }

// REJECT: Dynamic class construction (purging fails)
className={`bg-${status}-500`}

// CORRECT: Static lookup map
const statusClasses = {
  pending:   'bg-amber-100 text-amber-800 dark:bg-amber-900/30 dark:text-amber-300',
  confirmed: 'bg-blue-100 text-blue-800 dark:bg-blue-900/30 dark:text-blue-300',
  cancelled: 'bg-red-100 text-red-800 dark:bg-red-900/30 dark:text-red-300',
} as const;
```

---

## Lazy Loading Heavy Client Components

```tsx
// Use dynamic() for Client Components with heavy dependencies
import dynamic from 'next/dynamic';

// Chart library — don't load on server or in initial bundle
const OrdersChart = dynamic(
  () => import('@/components/orders/OrdersChart'),
  {
    loading: () => <ChartSkeleton />,
    ssr: false, // chart libraries are often browser-only
  }
);

// Rich text editor
const RichTextEditor = dynamic(
  () => import('@/components/ui/RichTextEditor'),
  {
    loading: () => <div className="h-40 animate-pulse rounded-lg bg-zinc-100 dark:bg-zinc-800" />,
    ssr: false,
  }
);
```

---

## Common Anti-Patterns

```tsx
// REJECT: Sync params access in Next.js 15
export default function Page({ params }: { params: { id: string } }) {
  const id = params.id; // runtime error in Next.js 15
}

// REJECT: 'use client' on page when only child needs it
'use client';
export default async function OrdersPage() { ... } // loses RSC benefits

// REJECT: Spinner in loading.tsx
export default function Loading() {
  return <div className="flex justify-center"><Spinner /></div>; // no layout hint
}

// REJECT: Server Action without auth
'use server';
export async function deleteOrder(id: string) {
  await db.order.delete({ where: { id } }); // no auth check!
}

// REJECT: Missing loading.tsx (content pops in without transition)
// Every route segment needs loading.tsx

// REJECT: Hardcoded cache in Next.js 15
const data = await fetch('/api/orders'); // cached by default in 14, NOT in 15
// USE: explicit { cache: 'no-store' } or { next: { revalidate: 60 } }

// REJECT: 'use client' on static components
'use client'; // not needed if no hooks/events
export function OrderStatusBadge({ status }: Props) {
  return <span>{status}</span>; // pure display — remove 'use client'
}
```

---

## Deployment Gates

- [ ] `next build` passes — zero errors, zero TS errors
- [ ] All dynamic API access (`params`, `searchParams`, `cookies`, `headers`) properly awaited
- [ ] Every route segment has `loading.tsx` with skeleton matching layout
- [ ] Every route segment has `error.tsx`
- [ ] All `loading.tsx` use skeleton components — no spinners
- [ ] All Server Actions validate input with Zod and check auth
- [ ] `'use client'` only on components that need interactivity — not on page files
- [ ] Heavy Client Components use `dynamic()` with skeleton fallback
- [ ] All pages export `metadata` or `generateMetadata`
- [ ] No secrets in `NEXT_PUBLIC_` env vars
- [ ] Core Web Vitals: LCP < 2.5s, CLS < 0.1, INP < 200ms
