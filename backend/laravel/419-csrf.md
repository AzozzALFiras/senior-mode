# Laravel — 419 CSRF & Token Handling Production Skill

## Role

You are a **senior Laravel security engineer** who understands every scenario that produces a 419 Page Expired error and has the exact fix for each one. 419 is never acceptable in production — it means your CSRF strategy is wrong for the use case.

---

## What Causes 419

| Scenario | Root Cause | Fix |
|----------|-----------|-----|
| Blade form missing `@csrf` | No token in form | Add `@csrf` |
| Session expired on long forms | Token tied to expired session | Extend session or notify user |
| SPA (React/Vue) hitting `/web` route | CSRF middleware on web routes | Use Sanctum SPA or API routes |
| API call to web route | Web route has CSRF protection | Move to `routes/api.php` |
| Inertia after session timeout | Inertia loses session | Handle in Inertia error handler |
| Multiple tabs / cache conflict | Stale token in old tab | Use `X-XSRF-TOKEN` cookie approach |
| CDN/proxy stripping cookies | `XSRF-TOKEN` cookie not reaching server | Configure `SESSION_DOMAIN` and `TRUSTED_PROXIES` |
| Mobile app hitting web route | No browser cookie support | Use Sanctum token auth on API routes |

---

## Fix 1 — Blade Forms

```blade
{{-- REQUIRED on every <form> that POSTs, PATCHes, PUTs, DELETEs --}}
<form method="POST" action="{{ route('orders.store') }}">
    @csrf
    @method('POST')  {{-- or 'PATCH', 'PUT', 'DELETE' --}}

    {{-- fields --}}

    <button type="submit">Submit</button>
</form>

{{-- REJECT: Missing @csrf --}}
<form method="POST" action="/orders">
    <input name="name" value="">
    <button>Submit</button>  {{-- 419 on submit --}}
</form>
```

---

## Fix 2 — Inertia.js (auto-handled, but session expiry needs care)

Inertia auto-manages CSRF via cookies — you do **not** add `@csrf` to Inertia forms. But session timeout causes 419:

```typescript
// resources/js/app.tsx — handle 419 globally in Inertia
import { router } from '@inertiajs/react';

router.on('invalid', (event) => {
    if (event.detail.response.status === 419) {
        // Session expired — reload the page to get fresh session/token
        event.preventDefault();
        if (confirm('Your session has expired. Reload the page?')) {
            window.location.reload();
        }
    }
});

// OR: In Inertia v2 — use error handling in app setup
createInertiaApp({
    resolve: (name) => resolvePageComponent(`./pages/${name}.vue`, import.meta.glob('./pages/**/*.vue')),
    setup({ el, App, props, plugin }) {
        return createApp({ render: () => h(App, props) })
            .use(plugin)
            .mount(el);
    },
});

// Handle in axios (if used alongside Inertia):
axios.interceptors.response.use(
    response => response,
    error => {
        if (error.response?.status === 419) {
            window.location.reload(); // get fresh CSRF token
        }
        return Promise.reject(error);
    }
);
```

---

## Fix 3 — SPA (React/Vue) + Laravel Sanctum

For a separate SPA hitting Laravel API:

```php
// config/cors.php — required for SPA
'paths'               => ['api/*', 'sanctum/csrf-cookie'],
'allowed_origins'     => [env('FRONTEND_URL', 'http://localhost:3000')],
'allowed_methods'     => ['*'],
'allowed_headers'     => ['*'],
'supports_credentials'=> true,  // CRITICAL — cookies won't work without this

// routes/web.php — add Sanctum cookie route
// This is auto-included with Sanctum
```

```typescript
// SPA: Initialize CSRF protection before first request
// Call this ONCE when the app boots
async function initCsrf() {
    await axios.get('/sanctum/csrf-cookie', { withCredentials: true });
}

// Configure axios globally
axios.defaults.withCredentials = true;
axios.defaults.withXSRFToken   = true;  // Axios 1.x — sends XSRF-TOKEN cookie as header

// .env on SPA side
VITE_API_URL=http://localhost:8000
FRONTEND_URL=http://localhost:3000  // on Laravel side
```

```php
// config/session.php — SPA on same domain
'domain'    => env('SESSION_DOMAIN', null),  // '.example.com' for subdomain sharing
'secure'    => env('SESSION_SECURE_COOKIE', true),
'same_site' => 'lax',  // 'none' only if cross-site with secure=true

// .env
SESSION_DOMAIN=.example.com  // allows api.example.com + app.example.com to share
```

---

## Fix 4 — REST API Routes (Stateless — no CSRF needed)

```php
// routes/api.php routes are EXEMPT from CSRF by default in Laravel
// API clients authenticate with Bearer tokens (Sanctum, Passport)
// CSRF is a browser-specific attack — stateless tokens are immune

// CORRECT: Mobile apps, third-party clients → routes/api.php with Bearer token
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/orders', [OrderController::class, 'index']);
});

// Client sends:
// Authorization: Bearer {token}
// NOT cookies — so CSRF doesn't apply

// REJECT: Mobile app hitting web route
// Route::post('/web/orders', ...) // has CSRF middleware — will 419
```

---

## Fix 5 — Proxy & Trusted Proxies (HTTPS + Load Balancer)

419 can happen when running behind nginx/load balancer because Laravel can't determine the correct origin:

```php
// app/Http/Middleware/TrustProxies.php (Laravel 10) 
// bootstrap/app.php (Laravel 11):
->withMiddleware(function (Middleware $middleware) {
    $middleware->trustProxies(
        at: '*',  // trust all proxies (for cloud environments)
        // OR: specific IPs:
        // at: ['10.0.0.1', '10.0.0.2'],
        headers: Request::HEADER_X_FORWARDED_FOR |
                 Request::HEADER_X_FORWARDED_HOST |
                 Request::HEADER_X_FORWARDED_PORT |
                 Request::HEADER_X_FORWARDED_PROTO
    );
})

// .env
APP_URL=https://example.com  // must match actual URL — affects cookie domain
```

---

## Fix 6 — Long-Running Forms (Session Timeout)

```php
// config/session.php
'lifetime' => 480,  // 8 hours for complex forms (was 120)
'expire_on_close' => false,

// For admin dashboards where users leave tabs open:
'lifetime' => 1440,  // 24 hours
```

```blade
{{-- Warn user before session expires --}}
<div
    x-data="{ warning: false }"
    x-init="setTimeout(() => warning = true, {{ (config('session.lifetime') - 5) * 60 * 1000 }})"
>
    <div x-show="warning" class="rounded-lg bg-amber-50 p-4 text-amber-800">
        Your session expires in 5 minutes. Save your work.
    </div>
</div>
```

---

## Fix 7 — AJAX / Fetch Requests in Blade Pages

```blade
{{-- Make CSRF token available to JS --}}
<meta name="csrf-token" content="{{ csrf_token() }}">
```

```javascript
// Vanilla JS — include token in every AJAX request
const token = document.querySelector('meta[name="csrf-token"]')?.content;

fetch('/orders', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-CSRF-TOKEN': token,  // required
        'Accept': 'application/json',
    },
    body: JSON.stringify(data),
});

// Axios — configure once globally in app.js
import axios from 'axios';
axios.defaults.headers.common['X-CSRF-TOKEN'] = document
    .querySelector('meta[name="csrf-token"]')?.content;
```

---

## Exception Handler — Friendly 419 Response

```php
// bootstrap/app.php — Laravel 11
->withExceptions(function (Exceptions $exceptions) {
    $exceptions->render(function (TokenMismatchException $e, Request $request) {
        if ($request->expectsJson()) {
            return response()->json([
                'message' => 'Session expired. Please refresh and try again.',
                'code'    => 'CSRF_TOKEN_MISMATCH',
            ], 419);
        }

        // For Blade: redirect back with error
        return redirect()->back()
            ->withInput()
            ->withErrors(['session' => 'Your session expired. Please try again.']);
    });
})
```

---

## Checklist — Debug a 419 in 60 Seconds

```
1. Is this a Blade form?
   → Check for @csrf inside <form>

2. Is this an Inertia form?
   → Don't add @csrf — use useForm() — check session hasn't expired

3. Is this an AJAX call from a Blade page?
   → Check: X-CSRF-TOKEN header present, meta csrf-token in <head>

4. Is this a SPA (separate frontend)?
   → Check: /sanctum/csrf-cookie called before first request
   → Check: withCredentials: true on all requests
   → Check: CORS supports_credentials: true
   → Check: SESSION_DOMAIN set correctly

5. Is this a mobile app / API client?
   → You're hitting the wrong route — use routes/api.php with Bearer token

6. Is this behind a load balancer?
   → Check: TrustProxies configured, APP_URL matches HTTPS URL

7. Is the session timing out?
   → Increase session.lifetime, add JS timeout warning
```
