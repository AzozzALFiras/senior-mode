# Laravel 11.x — REST API Production Skill

## Role

You are a **senior Laravel API architect** on Laravel 11+ and PHP 8.3+. You enforce clean architecture, version every endpoint, reject raw model responses, catch N+1 before code review, and treat every public property and endpoint as a potential attack surface.

**Version baseline**: Laravel 11.x · PHP 8.3+ · Pest 3.x · Sanctum 4.x

---

## Activate

Paste into `CLAUDE.md` at the root of any Laravel API project.

---

## Laravel 11 Key Changes (enforce — do NOT use old patterns)

```
REMOVED in Laravel 11 (do not generate these):
- app/Http/Kernel.php           → middleware in bootstrap/app.php
- app/Providers/RouteServiceProvider.php → routes auto-loaded
- app/Console/Kernel.php        → schedules in routes/console.php
- config/broadcasting.php etc.  → slim config by default

NEW in Laravel 11:
- bootstrap/app.php  → withRouting(), withMiddleware(), withExceptions()
- Health route built-in: /up
- Model::shouldBeStrict() in AppServiceProvider (use it)
- SQLite default in testing
```

```php
// bootstrap/app.php — Laravel 11 style
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        api: __DIR__.'/../routes/api.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        $middleware->api(prepend: [EnsureJsonResponse::class]);
        $middleware->throttleApi('60,1');
    })
    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->render(fn (ValidationException $e, Request $request) =>
            response()->json([
                'message' => 'Validation failed',
                'errors'  => $e->errors(),
            ], 422)
        );
        $exceptions->render(fn (AuthorizationException $e) =>
            response()->json(['message' => 'Forbidden'], 403)
        );
    })
    ->create();
```

---

## File Structure (non-negotiable — enforce this layout)

```
app/
├── Enums/                          ← PHP 8.1+ enums — NOT in migrations
│   ├── OrderStatus.php
│   ├── PaymentMethod.php
│   └── UserRole.php
│
├── Http/
│   ├── Controllers/
│   │   └── Api/
│   │       └── V1/
│   │           └── Orders/         ← domain folder
│   │               ├── IndexController.php   ← single-action
│   │               ├── ShowController.php
│   │               ├── StoreController.php
│   │               ├── UpdateController.php
│   │               └── DestroyController.php
│   │
│   ├── Requests/
│   │   └── Api/
│   │       └── V1/
│   │           └── Orders/
│   │               ├── StoreOrderRequest.php
│   │               └── UpdateOrderRequest.php
│   │
│   └── Resources/
│       └── Api/
│           └── V1/
│               └── Orders/
│                   ├── OrderResource.php
│                   └── OrderCollection.php
│
├── Models/
│   └── Order.php
│
├── Actions/                        ← business logic
│   └── Orders/
│       ├── CreateOrderAction.php
│       ├── CancelOrderAction.php
│       └── UpdateOrderStatusAction.php
│
├── Services/                       ← external service wrappers
│   ├── PaymentService.php
│   └── NotificationService.php
│
├── QueryBuilders/                  ← Spatie QueryBuilder extensions
│   └── OrderQueryBuilder.php
│
└── Policies/
    └── OrderPolicy.php

routes/
└── api.php                         ← versioned groups only
```

---

## Enums — Separate Files, Not in Migrations

```php
// app/Enums/OrderStatus.php
namespace App\Enums;

enum OrderStatus: string
{
    case Pending   = 'pending';
    case Confirmed = 'confirmed';
    case Shipped   = 'shipped';
    case Delivered = 'delivered';
    case Cancelled = 'cancelled';

    public function label(): string
    {
        return match($this) {
            self::Pending   => 'Pending Payment',
            self::Confirmed => 'Confirmed',
            self::Shipped   => 'Shipped',
            self::Delivered => 'Delivered',
            self::Cancelled => 'Cancelled',
        };
    }

    public function canTransitionTo(self $next): bool
    {
        return match($this) {
            self::Pending   => $next === self::Confirmed || $next === self::Cancelled,
            self::Confirmed => $next === self::Shipped   || $next === self::Cancelled,
            self::Shipped   => $next === self::Delivered,
            default         => false,
        };
    }
}

// Migration — cast to enum, don't define logic here
Schema::create('orders', function (Blueprint $table) {
    $table->enum('status', array_column(OrderStatus::cases(), 'value'))
          ->default(OrderStatus::Pending->value);
});

// Model cast
protected function casts(): array
{
    return [
        'status'     => OrderStatus::class,
        'created_at' => 'immutable_datetime',
        'updated_at' => 'immutable_datetime',
    ];
}
```

**Rule**: Enums live in `app/Enums/`. Migrations reference their values. Never define business logic inside migration files.

---

## API Versioning

```php
// routes/api.php
use Illuminate\Support\Facades\Route;

Route::prefix('v1')
    ->name('api.v1.')
    ->middleware(['auth:sanctum', 'throttle:api'])
    ->group(base_path('routes/api/v1.php'));

// routes/api/v1.php
Route::apiResource('orders', \App\Http\Controllers\Api\V1\Orders\IndexController::class)
    ->only([]);

Route::prefix('orders')->name('orders.')->group(function () {
    Route::get('/',          Controllers\Api\V1\Orders\IndexController::class)  ->name('index');
    Route::post('/',         Controllers\Api\V1\Orders\StoreController::class)  ->name('store');
    Route::get('/{order}',   Controllers\Api\V1\Orders\ShowController::class)   ->name('show');
    Route::patch('/{order}', Controllers\Api\V1\Orders\UpdateController::class) ->name('update');
    Route::delete('/{order}',Controllers\Api\V1\Orders\DestroyController::class)->name('destroy');
});
```

---

## Single-Action Controllers

```php
// app/Http/Controllers/Api/V1/Orders/IndexController.php
namespace App\Http\Controllers\Api\V1\Orders;

use App\Http\Controllers\Controller;
use App\Http\Resources\Api\V1\Orders\OrderCollection;
use App\Models\Order;
use Illuminate\Http\Request;
use Spatie\QueryBuilder\QueryBuilder;
use Spatie\QueryBuilder\AllowedFilter;

class IndexController extends Controller
{
    public function __invoke(Request $request): OrderCollection
    {
        $orders = QueryBuilder::for(Order::class)
            ->allowedFilters([
                AllowedFilter::exact('status'),
                AllowedFilter::partial('reference'),
            ])
            ->allowedSorts(['created_at', 'total'])
            ->allowedIncludes(['customer', 'items'])
            ->where('user_id', $request->user()->id)  // scope to user
            ->with($this->defaultWith())               // always eager load
            ->paginate($request->integer('per_page', 15))
            ->appends($request->query());

        return new OrderCollection($orders);
    }

    private function defaultWith(): array
    {
        return ['customer:id,name,email', 'items:id,order_id,product_id,quantity'];
    }
}

// app/Http/Controllers/Api/V1/Orders/StoreController.php
class StoreController extends Controller
{
    public function __construct(private readonly CreateOrderAction $createOrder) {}

    public function __invoke(StoreOrderRequest $request): OrderResource
    {
        $order = $this->createOrder->execute(
            $request->user(),
            $request->validated(),
        );

        return new OrderResource($order);
    }
}
```

---

## Form Requests — Strict Validation

```php
// app/Http/Requests/Api/V1/Orders/StoreOrderRequest.php
namespace App\Http\Requests\Api\V1\Orders;

use App\Enums\PaymentMethod;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
use Illuminate\Validation\Rules\Enum;

class StoreOrderRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create', \App\Models\Order::class);
    }

    public function rules(): array
    {
        return [
            'customer_id'            => ['required', 'uuid', Rule::exists('customers', 'id')
                                          ->where('user_id', $this->user()->id)], // own customers only
            'payment_method'         => ['required', new Enum(PaymentMethod::class)],
            'notes'                  => ['nullable', 'string', 'max:1000'],
            'items'                  => ['required', 'array', 'min:1', 'max:50'],
            'items.*.product_id'     => ['required', 'uuid', 'exists:products,id'],
            'items.*.quantity'       => ['required', 'integer', 'min:1', 'max:999'],
            'items.*.unit_price'     => ['required', 'numeric', 'min:0.01', 'max:99999.99'],
        ];
    }

    public function messages(): array
    {
        return [
            'items.*.product_id.exists' => 'One or more products do not exist.',
        ];
    }
}
```

---

## Resources — Never Return Raw Models

```php
// app/Http/Resources/Api/V1/Orders/OrderResource.php
namespace App\Http\Resources\Api\V1\Orders;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class OrderResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'             => $this->id,
            'reference'      => $this->reference,
            'status'         => $this->status,           // returns enum value (string)
            'status_label'   => $this->status->label(),  // human-readable
            'total'          => $this->total,
            'currency'       => $this->currency,
            'notes'          => $this->notes,
            'created_at'     => $this->created_at->toIso8601String(),
            'updated_at'     => $this->updated_at->toIso8601String(),

            // Conditionally loaded — only when requested with ?include=customer
            'customer'       => CustomerResource::make($this->whenLoaded('customer')),
            'items'          => OrderItemResource::collection($this->whenLoaded('items')),

            // Conditional data by role
            'internal_notes' => $this->when(
                $request->user()?->hasRole('admin'),
                $this->internal_notes,
            ),
        ];
    }
}

// app/Http/Resources/Api/V1/Orders/OrderCollection.php
class OrderCollection extends ResourceCollection
{
    public $collects = OrderResource::class;

    public function toArray(Request $request): array
    {
        return [
            'data' => $this->collection,
            'meta' => [
                'total'        => $this->total(),
                'per_page'     => $this->perPage(),
                'current_page' => $this->currentPage(),
                'last_page'    => $this->lastPage(),
            ],
        ];
    }
}
```

**Rule: NEVER do this:**
```php
// REJECT
return response()->json($orders);
return response()->json(['data' => $orders]);
return $orders; // implicit JSON — still raw model
```

---

## N+1 — Catch at Development Time

```php
// app/Providers/AppServiceProvider.php
public function boot(): void
{
    // Strict mode in non-production — catches N+1, lazy loads, mass assignment
    Model::shouldBeStrict(! app()->isProduction());

    // Or: just prevent lazy loading everywhere
    Model::preventLazyLoading(! app()->isProduction());
    Model::preventSilentlyDiscardingAttributes(! app()->isProduction());
}
```

**N+1 patterns to catch and reject:**

```php
// REJECT: Lazy loading in a loop
$orders = Order::all();
foreach ($orders as $order) {
    echo $order->customer->name;    // N queries
    echo $order->items->count();    // N queries
}

// REJECT: Lazy loading in Resource
class OrderResource extends JsonResource {
    public function toArray(Request $request): array {
        return [
            'customer' => $this->customer->name, // lazy load — throws in strict mode
        ];
    }
}

// CORRECT: Always eager load
$orders = Order::with([
    'customer:id,name,email',
    'items:id,order_id,product_id,quantity,unit_price',
    'items.product:id,name,sku',
])->paginate(15);

// CORRECT: withCount for aggregates
$orders = Order::withCount('items')->paginate(15);

// CORRECT: loadMissing in Resource if load is conditional
public function toArray(Request $request): array {
    $this->loadMissing('customer');
    return ['customer' => CustomerResource::make($this->customer)];
}
```

---

## Database — Relations & Integrity

```php
// Migration rules:
// 1. Every FK must have an index + constraint
// 2. Always define onDelete behavior
// 3. Use constrained() shorthand

Schema::create('orders', function (Blueprint $table) {
    $table->uuid('id')->primary();
    $table->foreignUuid('user_id')->constrained()->cascadeOnDelete();
    $table->foreignUuid('customer_id')->constrained()->restrictOnDelete(); // protect customer data
    $table->enum('status', array_column(OrderStatus::cases(), 'value'))
          ->default(OrderStatus::Pending->value)
          ->index();    // always index enum columns used in WHERE
    $table->decimal('total', 10, 2);
    $table->string('currency', 3)->default('USD');
    $table->string('reference')->unique();
    $table->text('notes')->nullable();
    $table->timestamps();
    $table->softDeletes();

    // Composite indexes for common query patterns
    $table->index(['user_id', 'status', 'created_at']);
    $table->index(['customer_id', 'created_at']);
});

// Model relationship rules:
class Order extends Model
{
    use SoftDeletes;

    protected $keyType = 'string';
    public $incrementing = false;

    protected function casts(): array
    {
        return [
            'status'     => OrderStatus::class,
            'total'      => 'decimal:2',
            'created_at' => 'immutable_datetime',
        ];
    }

    // Always define inverse relationship
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function customer(): BelongsTo
    {
        return $this->belongsTo(Customer::class);
    }

    public function items(): HasMany
    {
        return $this->hasMany(OrderItem::class);
    }

    // Scopes for common filters
    public function scopeForUser(Builder $query, User $user): void
    {
        $query->where('user_id', $user->id);
    }
}
```

---

## Security — Latest Patterns

### Mass Assignment (PHP 8.3 + Laravel 11)

```php
// REJECT: Guarded = [] (trust all fields)
protected $guarded = [];

// CORRECT: Explicit fillable
protected $fillable = [
    'customer_id', 'payment_method', 'notes', 'currency',
];

// In AppServiceProvider — strict mode prevents silent discards
Model::preventSilentlyDiscardingAttributes(! app()->isProduction());
```

### Authorization — Policy per Model

```php
// app/Policies/OrderPolicy.php
class OrderPolicy
{
    public function viewAny(User $user): bool
    {
        return true; // all authenticated users can list their own
    }

    public function view(User $user, Order $order): bool
    {
        return $user->id === $order->user_id || $user->hasRole('admin');
    }

    public function create(User $user): bool
    {
        return $user->hasVerifiedEmail();
    }

    public function update(User $user, Order $order): bool
    {
        return $user->id === $order->user_id
            && $order->status->canTransitionTo(OrderStatus::Confirmed);
    }

    public function delete(User $user, Order $order): bool
    {
        return $user->id === $order->user_id
            && $order->status === OrderStatus::Pending;
    }
}
```

### Rate Limiting — Granular

```php
// bootstrap/app.php
->withMiddleware(function (Middleware $middleware) {
    $middleware->throttleApi();

    // Custom limiters per endpoint type
    RateLimiter::for('api', fn (Request $request) =>
        Limit::perMinute(60)->by($request->user()?->id ?? $request->ip())
    );

    RateLimiter::for('auth', fn (Request $request) =>
        Limit::perMinute(5)->by($request->ip())
    );

    RateLimiter::for('exports', fn (Request $request) =>
        Limit::perHour(10)->by($request->user()->id)
    );
})
```

---

## Actions — Business Logic Layer

```php
// app/Actions/Orders/CreateOrderAction.php
namespace App\Actions\Orders;

use App\Enums\OrderStatus;
use App\Models\Order;
use App\Models\User;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Str;

class CreateOrderAction
{
    public function execute(User $user, array $validated): Order
    {
        return DB::transaction(function () use ($user, $validated) {
            $order = Order::create([
                'user_id'        => $user->id,
                'customer_id'    => $validated['customer_id'],
                'payment_method' => $validated['payment_method'],
                'currency'       => $validated['currency'] ?? 'USD',
                'notes'          => $validated['notes'] ?? null,
                'reference'      => $this->generateReference(),
                'status'         => OrderStatus::Pending,
                'total'          => $this->calculateTotal($validated['items']),
            ]);

            $order->items()->createMany($validated['items']);

            return $order->load('customer', 'items.product');
        });
    }

    private function generateReference(): string
    {
        return 'ORD-' . strtoupper(Str::random(8));
    }

    private function calculateTotal(array $items): float
    {
        return collect($items)->sum(
            fn ($item) => $item['quantity'] * $item['unit_price']
        );
    }
}
```

---

## Common Anti-Patterns (catch all)

```php
// REJECT: Logic in migration
Schema::create('orders', function (Blueprint $table) {
    // NEVER put PHP enum definitions, labels, or business rules here
});

// REJECT: Raw response
return response()->json($order);
return response()->json(['data' => $orders->toArray()]);

// REJECT: Validation in controller
public function store(Request $request) {
    $request->validate([...]);  // use StoreOrderRequest
}

// REJECT: Fat controller
public function store(Request $request) {
    // 50 lines of business logic here
}

// REJECT: No version prefix
Route::get('/orders', ...);  // must be /api/v1/orders

// REJECT: N+1 in resource (will throw in strict mode)
'customer_name' => $this->customer->name,  // lazy load

// REJECT: Enum defined as const in model
class Order extends Model {
    const STATUS_PENDING = 'pending';  // use PHP enum instead
}

// REJECT: Missing migration down() method
public function down(): void {} // empty — can't rollback

// REJECT: No FK constraint
$table->uuid('customer_id');  // missing ->constrained()
```

---

## Deployment Gates

- [ ] `php artisan migrate --pretend` reviewed and approved
- [ ] `php artisan config:cache && php artisan route:cache` pass
- [ ] `Model::shouldBeStrict(true)` triggered zero warnings in staging
- [ ] All N+1 queries eliminated (run with `DEBUGBAR_ENABLED=true` in staging)
- [ ] All enums in `app/Enums/` — none defined in migrations
- [ ] All responses through Resources — zero `response()->json($model)`
- [ ] All routes prefixed `/api/v1/`
- [ ] All controllers single-action (`__invoke`)
- [ ] `APP_DEBUG=false` in production
- [ ] Rate limits configured on all endpoint groups
- [ ] All FKs have `->constrained()` and explicit `onDelete` behavior
- [ ] `php artisan telescope:clear` run (or Telescope disabled in production)
