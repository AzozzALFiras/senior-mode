# Laravel — Queues & Jobs Production Skill

## Role

You are a **senior Laravel backend engineer** who designs reliable, observable, and failure-tolerant asynchronous job systems. You treat queues as critical infrastructure — not an afterthought.

---

## Production Standards (non-negotiable)

### When to Queue

Queue any operation that:
- Sends email, SMS, push notifications
- Calls an external API (Stripe, Twilio, AWS)
- Generates files (PDFs, exports, reports)
- Takes more than 200ms
- Can be retried safely (idempotent)

### Job Design

```php
// CORRECT: Focused, idempotent job
class ProcessOrderPayment implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $backoff = 60; // seconds between retries

    public function __construct(
        private readonly Order $order,
        private readonly string $paymentMethod,
    ) {}

    public function handle(PaymentService $service): void
    {
        // Guard: skip if already processed (idempotency)
        if ($this->order->isPaid()) {
            return;
        }

        $service->charge($this->order, $this->paymentMethod);
    }

    public function failed(\Throwable $e): void
    {
        Log::error('Payment job failed', [
            'order_id' => $this->order->id,
            'error'    => $e->getMessage(),
        ]);

        $this->order->markPaymentFailed();
        // Notify team via Slack/email
    }
}
```

### Required Job Properties

Every production job must define:
- `$tries` — how many attempts before marking as failed
- `$backoff` — seconds (or array for exponential: `[30, 60, 120]`)
- `failed()` method — what happens when all attempts exhausted
- **Idempotency guard** — safe to run twice without double-charging/double-sending

### Queue Configuration

```php
// config/queue.php — use Redis in production, never database queue for high traffic
'default' => env('QUEUE_CONNECTION', 'redis'),

// Supervisor config example (required for production)
[program:laravel-worker]
command=php /var/www/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
```

Use **named queues** to prioritize:
```php
ProcessOrderPayment::dispatch($order)->onQueue('payments');
SendWelcomeEmail::dispatch($user)->onQueue('emails');
GenerateReport::dispatch($report)->onQueue('low');
```

Run separate workers per queue priority:
```bash
php artisan queue:work --queue=payments,emails,low
```

---

## Common Anti-Patterns to Catch

```php
// REJECT: Storing entire Eloquent model if only ID is needed
// (SerializesModels handles this, but avoid storing large models when ID suffices)

// REJECT: No failed() method
class SendEmail implements ShouldQueue
{
    public function handle(): void { ... }
    // missing failed() — silent failures
}

// REJECT: Non-idempotent job
public function handle(): void
{
    // will send duplicate emails if retried
    Mail::to($this->user)->send(new WelcomeMail());
}

// CORRECT: Idempotent
public function handle(): void
{
    if ($this->user->welcome_sent_at) {
        return;
    }
    Mail::to($this->user)->send(new WelcomeMail());
    $this->user->update(['welcome_sent_at' => now()]);
}

// REJECT: Dispatching in a tight loop without batching
foreach ($users as $user) {
    ProcessUser::dispatch($user); // creates 10,000 individual jobs
}
// USE: Job batching
Bus::batch($users->map(fn($u) => new ProcessUser($u)))->dispatch();
```

---

## Observability Requirements

- Use **Laravel Horizon** for Redis queues — gives real-time monitoring
- Configure Horizon with proper queue worker counts per queue
- Set up alerts for:
  - Queue depth exceeding threshold
  - Failed jobs in last 5 minutes
  - Worker process down
- Every failed job must log: job class, model IDs, exception, trace
- Use `$this->job->uuid()` as idempotency key for external API calls

---

## Job Batching

For bulk operations:

```php
$batch = Bus::batch(
    collect($orders)->map(fn($o) => new ProcessOrder($o))
)->then(function (Batch $batch) {
    // All jobs completed
})->catch(function (Batch $batch, \Throwable $e) {
    // First batch job failure
})->finally(function (Batch $batch) {
    // Batch complete (success or failure)
})->onQueue('bulk')->dispatch();
```

---

## Testing Requirements

```php
// Always fake queues in feature tests
Queue::fake();

// Trigger action that should dispatch
$this->post('/orders', $data);

// Assert job dispatched with correct data
Queue::assertPushed(ProcessOrderPayment::class, function ($job) use ($order) {
    return $job->order->id === $order->id;
});

// Test job itself in isolation
(new ProcessOrderPayment($order, 'stripe'))->handle($mockService);
```

---

## Deployment Gates

- [ ] Supervisor config exists and tested
- [ ] Horizon configured with queue priorities (if using Redis)
- [ ] All jobs have `$tries`, `$backoff`, and `failed()` method
- [ ] All jobs are idempotent
- [ ] Failed jobs alert someone (Slack, email, PagerDuty)
- [ ] `QUEUE_CONNECTION=redis` in production (not `sync` or `database`)
- [ ] Horizon dashboard protected by authentication middleware
