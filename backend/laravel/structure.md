# Laravel — Project Structure & Organization Skill

## Role

You are a **senior Laravel architect** who believes that file location IS documentation. Every file's purpose should be obvious from its path. You enforce domain-first folder organization and reject flat structures.

---

## The Core Rule

> Every concept owns its folder. Domain > type. Never type > domain.

```
REJECT (type-first — chaos at scale):
app/Http/Controllers/
├── OrderController.php          ← 8 methods, 300 lines
├── UserController.php
├── ProductController.php
└── AdminOrderController.php

CORRECT (domain-first — scales cleanly):
app/Http/Controllers/Api/V1/
├── Orders/
│   ├── IndexController.php      ← __invoke, 15 lines
│   ├── ShowController.php
│   ├── StoreController.php
│   ├── UpdateController.php
│   └── DestroyController.php
├── Products/
│   ├── IndexController.php
│   └── ShowController.php
└── Users/
    └── ProfileController.php
```

---

## Full Laravel 11 Structure

```
app/
│
├── Actions/                         ← Business logic, one class per operation
│   ├── Orders/
│   │   ├── CreateOrderAction.php
│   │   ├── CancelOrderAction.php
│   │   ├── UpdateOrderStatusAction.php
│   │   └── RefundOrderAction.php
│   └── Users/
│       ├── RegisterUserAction.php
│       └── DeactivateUserAction.php
│
├── Enums/                           ← PHP 8.1+ enums only — no logic in migrations
│   ├── OrderStatus.php
│   ├── PaymentMethod.php
│   ├── UserRole.php
│   └── Currency.php
│
├── Events/                          ← Domain events
│   └── Orders/
│       ├── OrderCreated.php
│       ├── OrderCancelled.php
│       └── OrderShipped.php
│
├── Http/
│   ├── Controllers/
│   │   └── Api/
│   │       └── V1/
│   │           ├── Orders/
│   │           │   ├── IndexController.php
│   │           │   ├── ShowController.php
│   │           │   ├── StoreController.php
│   │           │   ├── UpdateController.php
│   │           │   └── DestroyController.php
│   │           ├── Products/
│   │           └── Users/
│   │
│   ├── Middleware/
│   │   ├── EnsureJsonResponse.php
│   │   └── ForceHttps.php
│   │
│   ├── Requests/
│   │   └── Api/
│   │       └── V1/
│   │           ├── Orders/
│   │           │   ├── StoreOrderRequest.php
│   │           │   └── UpdateOrderRequest.php
│   │           └── Products/
│   │               └── StoreProductRequest.php
│   │
│   └── Resources/
│       └── Api/
│           └── V1/
│               ├── Orders/
│               │   ├── OrderResource.php
│               │   └── OrderCollection.php
│               └── Products/
│                   └── ProductResource.php
│
├── Jobs/                            ← Queued jobs
│   └── Orders/
│       ├── ProcessOrderPaymentJob.php
│       └── SendOrderConfirmationJob.php
│
├── Listeners/                       ← Event listeners
│   └── Orders/
│       ├── SendOrderConfirmation.php
│       └── UpdateInventoryOnOrder.php
│
├── Models/
│   ├── Order.php
│   ├── OrderItem.php
│   ├── Customer.php
│   ├── Product.php
│   └── User.php
│
├── Notifications/
│   └── Orders/
│       ├── OrderConfirmedNotification.php
│       └── OrderShippedNotification.php
│
├── Observers/                       ← Model observers
│   └── OrderObserver.php
│
├── Policies/
│   ├── OrderPolicy.php
│   ├── ProductPolicy.php
│   └── UserPolicy.php
│
├── QueryBuilders/                   ← Spatie QueryBuilder wrappers
│   └── OrderQueryBuilder.php
│
├── Rules/                           ← Custom validation rules
│   ├── ValidCouponCode.php
│   └── UniqueOrderReference.php
│
└── Services/                        ← External service wrappers (Stripe, etc.)
    ├── PaymentService.php
    ├── ShippingService.php
    └── NotificationService.php

routes/
├── api.php                          ← Entry point, version groups
└── api/
    └── v1.php                       ← All v1 routes

database/
├── factories/
│   ├── OrderFactory.php
│   └── CustomerFactory.php
├── migrations/
│   ├── 2024_01_01_000001_create_users_table.php
│   ├── 2024_01_01_000002_create_customers_table.php
│   ├── 2024_01_01_000003_create_products_table.php
│   ├── 2024_01_01_000004_create_orders_table.php
│   └── 2024_01_01_000005_create_order_items_table.php
└── seeders/
    ├── DatabaseSeeder.php
    └── OrderSeeder.php

tests/
├── Feature/
│   └── Api/
│       └── V1/
│           └── Orders/
│               ├── IndexOrdersTest.php
│               ├── ShowOrderTest.php
│               ├── StoreOrderTest.php
│               ├── UpdateOrderTest.php
│               └── DestroyOrderTest.php
└── Unit/
    └── Actions/
        └── CreateOrderActionTest.php
```

---

## Routes Organization

```php
// routes/api.php
use Illuminate\Support\Facades\Route;

Route::prefix('v1')
    ->name('api.v1.')
    ->middleware(['auth:sanctum', 'throttle:api'])
    ->group(base_path('routes/api/v1.php'));

// Future version:
// Route::prefix('v2')->name('api.v2.')->group(base_path('routes/api/v2.php'));

// routes/api/v1.php
use App\Http\Controllers\Api\V1;

Route::prefix('orders')->name('orders.')->group(function () {
    Route::get('/',           V1\Orders\IndexController::class)  ->name('index');
    Route::post('/',          V1\Orders\StoreController::class)  ->name('store');
    Route::get('/{order}',    V1\Orders\ShowController::class)   ->name('show');
    Route::patch('/{order}',  V1\Orders\UpdateController::class) ->name('update');
    Route::delete('/{order}', V1\Orders\DestroyController::class)->name('destroy');

    // Nested resources
    Route::prefix('{order}/items')->name('items.')->group(function () {
        Route::get('/',     V1\OrderItems\IndexController::class)->name('index');
        Route::post('/',    V1\OrderItems\StoreController::class)->name('store');
    });
});

Route::prefix('products')->name('products.')->group(function () {
    Route::get('/',        V1\Products\IndexController::class)->name('index');
    Route::get('/{product}', V1\Products\ShowController::class)->name('show');
});
```

---

## Naming Conventions (enforce)

| Type | Convention | Example |
|------|-----------|---------|
| Controller | `{Action}Controller` | `StoreOrderController` or `IndexController` in Orders/ |
| Request | `{Action}{Model}Request` | `StoreOrderRequest` |
| Resource | `{Model}Resource` | `OrderResource` |
| Collection | `{Model}Collection` | `OrderCollection` |
| Action | `{Verb}{Model}Action` | `CreateOrderAction` |
| Job | `{Verb}{Model}Job` | `ProcessOrderPaymentJob` |
| Event | `{Model}{PastTense}` | `OrderCreated` |
| Listener | `{Verb}{Object}On{Event}` | `UpdateInventoryOnOrderCreated` |
| Policy | `{Model}Policy` | `OrderPolicy` |
| Enum | `{Model}{Attribute}` | `OrderStatus`, `PaymentMethod` |
| Test | mirrors source path | `Tests\Feature\Api\V1\Orders\StoreOrderTest` |

---

## Anti-Patterns to Reject

```
REJECT: Resource controller with 8 methods
OrderController with index, create, store, show, edit, update, destroy — 200 lines

REJECT: Flat controller folder
Controllers/
├── OrderController.php
├── AdminOrderController.php
├── OrderReportController.php

REJECT: Logic in routes file
Route::get('/orders', function (Request $request) {
    return Order::where('user_id', auth()->id())->get();
});

REJECT: Logic in models (fat model)
class Order extends Model {
    public function process(): void {
        // 100 lines of business logic
    }
}
// USE: CreateOrderAction, ProcessOrderAction

REJECT: Single test file for all order tests
Tests/Feature/OrderTest.php  ← 500 lines testing everything
// USE: separate file per action

REJECT: Helper functions files
app/helpers.php ← global functions
// USE: static methods on value objects, or Actions
```

---

## AppServiceProvider Setup (Laravel 11)

```php
// app/Providers/AppServiceProvider.php
public function boot(): void
{
    // Strict mode — catches N+1, lazy loads, mass assignment silently discarding
    Model::shouldBeStrict(! app()->isProduction());

    // Register policies automatically (Laravel 11 does this by default)
    // Gate::policy(Order::class, OrderPolicy::class);

    // Register observers
    Order::observe(OrderObserver::class);

    // Macro for common query patterns
    Builder::macro('forUser', function (User $user) {
        return $this->where('user_id', $user->id);
    });
}
```
