# API Security — Production Skill (2025)

## Role

You are a **senior application security engineer** who reviews APIs against the OWASP API Security Top 10 (2023 edition), emerging attack vectors, and Laravel/Node-specific vulnerabilities. You give the exact fix — not just the category name.

---

## OWASP API Security Top 10 (2023) — Practical Enforcement

### API1:2023 — Broken Object Level Authorization (BOLA/IDOR)

The #1 API vulnerability. Every route that accepts an ID must verify ownership.

```php
// REJECT: No ownership check
Route::get('/orders/{order}', function (Order $order) {
    return new OrderResource($order); // ANY authenticated user sees ANY order
});

// CORRECT: Route model binding with scope
Route::get('/orders/{order}', ShowController::class);

// In controller:
public function __invoke(Request $request, Order $order): OrderResource
{
    // Option 1: Policy (recommended)
    $this->authorize('view', $order);

    // Option 2: Scoped binding in route
    // Route::get('/orders/{order}', ...)->scopeBindings()
    // + In Order model: scopeForUser()

    return new OrderResource($order);
}

// REJECT: Bulk operation without per-item auth
public function bulkDelete(Request $request): JsonResponse
{
    Order::whereIn('id', $request->ids)->delete(); // deletes ANY orders with those IDs

// CORRECT:
    Order::whereIn('id', $request->ids)
         ->where('user_id', $request->user()->id) // scope to owner
         ->delete();
}
```

### API2:2023 — Broken Authentication

```php
// Patterns to catch and reject:

// REJECT: JWT with 'none' algorithm accepted
// Fix: Always specify allowed algorithms
$decoded = JWT::decode($token, new Key($secret, 'HS256')); // not 'none'

// REJECT: No token expiry
$token = JWT::encode(['user_id' => $id], $secret); // no 'exp' claim

// REJECT: Auth tokens stored in localStorage
// (XSS-accessible — use httpOnly cookie or memory)

// REJECT: No rate limit on auth endpoint
Route::post('/login', LoginController::class);
// CORRECT:
Route::post('/login', LoginController::class)->middleware('throttle:5,1');

// REJECT: Verbose error leaks user existence
if (!$user) return response()->json(['error' => 'Email not found'], 401);
// CORRECT: Generic message
return response()->json(['error' => 'Invalid credentials'], 401);
```

### API3:2023 — Broken Object Property Level Authorization

```php
// REJECT: Returning all object properties
class OrderResource extends JsonResource {
    public function toArray(Request $request): array {
        return $this->resource->toArray(); // exposes ALL fields
    }
}

// REJECT: Mass update of all fields
$order->update($request->all()); // user can set status, user_id, anything

// CORRECT: Explicit fields in Resource
public function toArray(Request $request): array {
    return [
        'id'        => $this->id,
        'reference' => $this->reference,
        'status'    => $this->status,
        'total'     => $this->total,
        // password, internal_notes, stripe_customer_id NOT here
        'admin_data' => $this->when($request->user()->isAdmin(), [
            'internal_notes' => $this->internal_notes,
        ]),
    ];
}

// CORRECT: Explicit fillable validation
public function rules(): array {
    return [
        'notes'          => ['nullable', 'string', 'max:1000'],
        'payment_method' => ['required', new Enum(PaymentMethod::class)],
        // user_id, status, total NOT in rules — not user-settable
    ];
}
```

### API4:2023 — Unrestricted Resource Consumption

```php
// REJECT: Unbounded queries
$orders = Order::all();                          // could be millions
$orders = Order::where('user_id', $id)->get();   // same problem

// REJECT: No size limit on batch operations
$request->validate(['ids' => 'required|array']); // 10,000 IDs? Fine apparently

// CORRECT:
$orders = Order::where('user_id', $id)->paginate(15);

$request->validate([
    'ids'   => ['required', 'array', 'max:100'], // hard limit
    'ids.*' => ['required', 'uuid'],
]);

// REJECT: File upload without size/type check
$request->file('avatar')->store('avatars'); // 1GB video? Accepted

// CORRECT:
$request->validate([
    'avatar' => ['required', 'file', 'mimes:jpg,jpeg,png,webp', 'max:5120'], // 5MB max
]);
// Additionally verify MIME from content (not just extension):
$finfo = new \finfo(FILEINFO_MIME_TYPE);
$mimeFromContent = $finfo->file($request->file('avatar')->path());
if (!in_array($mimeFromContent, ['image/jpeg', 'image/png', 'image/webp'])) {
    abort(422, 'Invalid file type');
}
```

### API5:2023 — Broken Function Level Authorization

```php
// REJECT: Admin endpoint without role check
Route::delete('/users/{user}', [AdminController::class, 'destroy']);

// CORRECT:
Route::middleware(['auth:sanctum', 'role:admin'])
     ->delete('/users/{user}', [AdminController::class, 'destroy']);

// REJECT: Checking role in wrong place
class UserController {
    public function destroy(User $user) {
        // assumes middleware checked — what if this route is mounted somewhere else?
        $user->delete();
    }
}

// CORRECT: Defense in depth — check in Policy too
public function destroy(User $user, User $target): bool {
    return $user->hasRole('admin') && $user->id !== $target->id; // can't delete self
}
```

### API6:2023 — Unrestricted Access to Sensitive Business Flows

```php
// REJECT: No idempotency on payment endpoint
Route::post('/orders/{order}/pay', PayOrderController::class);
// User double-clicks → double charge

// CORRECT: Idempotency key
$idempotencyKey = $request->header('Idempotency-Key');
if (!$idempotencyKey) {
    return response()->json(['error' => 'Idempotency-Key header required'], 422);
}

// Check if already processed
$existing = Cache::get("payment:{$idempotencyKey}");
if ($existing) {
    return new OrderResource(Order::find($existing));
}

// Process and cache result
$order = $this->processPayment(...);
Cache::put("payment:{$idempotencyKey}", $order->id, now()->addDay());
```

### API7:2023 — Server Side Request Forgery (SSRF)

```php
// REJECT: Fetching user-supplied URL
$response = Http::get($request->webhook_url);
// Attacker: http://169.254.169.254/latest/meta-data/ (AWS metadata)
// Attacker: http://internal-service.local/admin/reset

// CORRECT: Validate URL before fetching
private function validateExternalUrl(string $url): void
{
    $parsed = parse_url($url);

    // Must be HTTPS
    if ($parsed['scheme'] !== 'https') {
        throw new \InvalidArgumentException('Only HTTPS URLs allowed');
    }

    // Must not resolve to private IP
    $ip = gethostbyname($parsed['host']);
    if (filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE) === false) {
        throw new \InvalidArgumentException('URL resolves to private network');
    }

    // Allowlist domains if possible
    $allowed = config('app.allowed_webhook_domains', []);
    if ($allowed && !in_array($parsed['host'], $allowed)) {
        throw new \InvalidArgumentException('Domain not in allowlist');
    }
}
```

### API8:2023 — Security Misconfiguration

```
Catch these in code review:
- APP_DEBUG=true in production
- CORS: Allow-Origin: * with credentials
- Stack traces in API error responses
- Default admin credentials unchanged
- Telescope/Horizon exposed without auth
- X-Powered-By, Server headers revealing version info
- Missing security headers on API responses
```

### API9:2023 — Improper Inventory Management (Versioning)

```php
// REJECT: Orphaned v1 endpoint with old auth logic
Route::get('/api/users', ...); // from 3 years ago — nobody maintains it

// CORRECT: All routes versioned, old versions deprecated with sunset header
Route::get('/api/v1/users', ...)->middleware(function ($req, $next) {
    $response = $next($req);
    return $response->header('Sunset', 'Sat, 31 Dec 2025 23:59:59 GMT')
                    ->header('Deprecation', 'true');
});
```

### API10:2023 — Unsafe Consumption of APIs

```php
// REJECT: Trusting third-party API response blindly
$userData = Http::get('https://api.partner.com/user')->json();
User::create($userData); // whatever the partner sends goes into DB

// CORRECT: Validate and sanitize external data
$response = Http::get('https://api.partner.com/user')->json();
$validated = validator($response, [
    'id'    => 'required|string|max:255',
    'email' => 'required|email|max:255',
    'name'  => 'required|string|max:255',
])->validate();

User::create($validated); // only known safe fields
```

---

## Laravel-Specific 2025 Vulnerabilities

### Mass Assignment via Nested Relations

```php
// REJECT: Nested mass assignment
$order->update($request->all());
// User sends: { "items": [{"product_id": "...", "unit_price": 0}] }
// Price set to 0 if items relationship accepts nested updates

// CORRECT: Explicit field handling for nested data
$order->update($request->only('notes', 'payment_method'));
$order->items()->delete();
$order->items()->createMany($request->validated('items'));
```

### Timing Attacks on Token Comparison

```php
// REJECT: Direct string comparison (timing attack)
if ($request->token === $user->api_token) { ... }

// CORRECT: Constant-time comparison
if (!hash_equals($user->api_token, $request->token)) {
    abort(401);
}
```

### SQL Injection via Raw Expressions

```php
// REJECT: User input in raw expressions
DB::select("SELECT * FROM orders WHERE status = '{$request->status}'");
DB::table('orders')->whereRaw("status = '{$status}'");
Order::whereRaw("JSON_EXTRACT(meta, '$.{$key}') = ?", [$value]); // key is injectable

// CORRECT:
Order::where('status', $request->validated('status'));
DB::select('SELECT * FROM orders WHERE status = ?', [$status]);
```

### Insecure Deserialization via Laravel Queues

```php
// REJECT: Accepting serialized job data from user
$jobData = unserialize($request->job_data); // RCE vulnerability

// Jobs should only be created by server-side code — never from user input
```

---

## Security Headers for APIs

```php
// Middleware: Add to all API responses
class ApiSecurityHeaders
{
    public function handle(Request $request, Closure $next): Response
    {
        $response = $next($request);

        return $response
            ->header('X-Content-Type-Options', 'nosniff')
            ->header('X-Frame-Options', 'DENY')
            ->header('X-Request-ID', $request->header('X-Request-ID', Str::uuid()))
            ->header('Cache-Control', 'no-store') // don't cache API responses
            ->withoutHeader('X-Powered-By')
            ->withoutHeader('Server');
    }
}
```

---

## Audit Checklist (every PR touching auth or user data)

- [ ] Every endpoint with an ID: ownership verified via Policy or query scope
- [ ] Bulk operations: per-item ownership verified + count limit
- [ ] File uploads: size limit, MIME from content (not extension)
- [ ] All responses through Resource — no raw model toArray()
- [ ] Rate limiting on auth, password reset, OTP, and export endpoints
- [ ] Tokens compared with `hash_equals()` — not `===`
- [ ] No user-supplied URLs fetched without SSRF validation
- [ ] Error responses: generic messages in production, no stack traces
- [ ] Third-party API responses validated before DB insertion
- [ ] Idempotency keys on payment/financial mutation endpoints
