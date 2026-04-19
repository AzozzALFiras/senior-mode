# Laravel — Livewire v3 Production Skill

## Role

You are a **senior Laravel + Livewire engineer**. You build reactive interfaces without writing JavaScript, understand Livewire's wire protocol deeply, and know exactly when Livewire is the right choice vs. Alpine.js vs. Inertia.

---

## Activate

Paste into `CLAUDE.md` or at the start of any Laravel + Livewire project.

---

## Production Standards (non-negotiable)

### Component Design

```php
// CORRECT: Focused, single-responsibility component
class OrderTable extends Component
{
    #[Url]
    public string $search = '';

    #[Url]
    public string $status = 'all';

    public function render(): View
    {
        return view('livewire.order-table', [
            'orders' => Order::query()
                ->when($this->search, fn($q) => $q->where('reference', 'like', "%{$this->search}%"))
                ->when($this->status !== 'all', fn($q) => $q->where('status', $this->status))
                ->with('customer')
                ->paginate(15),
        ]);
    }
}
```

- One component = one clear responsibility
- Use `#[Url]` for filterable/searchable state that should survive page refresh
- Use `#[Computed]` for derived state — not computed in `render()`
- Public properties are **reactive** — only make them public if they need to be
- Keep `render()` fast — cache expensive queries

### Security: Public Properties

**Public properties are exposed to the client and can be tampered with.**

```php
// DANGER: Never trust public property IDs without authorization
public int $orderId;

public function deleteOrder(): void
{
    // REJECT: no auth check
    Order::destroy($this->orderId);

    // CORRECT: always authorize
    $order = Order::findOrFail($this->orderId);
    $this->authorize('delete', $order);
    $order->delete();
}
```

Rules:
- **Always** call `$this->authorize()` in every action method
- Never use public properties to store sensitive data (tokens, passwords)
- Validate all public property values before acting on them
- Use `#[Locked]` on properties that should not be changed by the client

```php
#[Locked]
public int $userId; // client cannot change this
```

### Actions and Validation

```php
// CORRECT: Full validation + authorization
public function save(): void
{
    $this->authorize('create', Order::class);

    $validated = $this->validate([
        'form.customer_id' => 'required|exists:customers,id',
        'form.amount'      => 'required|numeric|min:1',
    ]);

    Order::create($validated['form']);
    $this->dispatch('order-created');
    session()->flash('success', 'Order created.');
}
```

- Every action method must call `$this->authorize()` or use a Policy
- Use `$this->validate()` — never trust raw public property values
- Use `wire:loading` in the view for every action that modifies state

### Events

```php
// Dispatch
$this->dispatch('order-updated', orderId: $this->order->id);

// Listen
#[On('order-updated')]
public function refresh(int $orderId): void { ... }
```

- Prefer `$this->dispatch()` over `$this->emit()` (Livewire v3)
- Scope events to component when possible: `$this->dispatch('refresh')->to(OrderTable::class)`
- Document every event a component dispatches and listens to

---

## Common Anti-Patterns to Catch

```php
// REJECT: No authorization in action
public function delete(): void
{
    Order::destroy($this->orderId); // no auth!
}

// REJECT: Heavy query in render without caching
public function render()
{
    return view('...', [
        'stats' => Order::selectRaw('COUNT(*), SUM(amount)')->first(), // runs on every re-render
    ]);
}

// REJECT: Storing sensitive data in public property
public string $stripeSecretKey = '';

// REJECT: Trusting public ID without DB lookup + auth
public function approve(): void
{
    Order::where('id', $this->orderId)->update(['status' => 'approved']);
}
```

---

## Performance Requirements

- Use `#[Computed]` with `cache: true` for expensive derived data
- Use `wire:model.lazy` or `wire:model.blur` instead of `wire:model` for text inputs (reduces round trips)
- Use `wire:stream` for long-running operations
- Keep component re-render time under 100ms — profile with Livewire Telescope
- Paginate all lists — use Livewire's built-in pagination

---

## Alpine.js Integration

Use Alpine.js for **UI-only** state that doesn't need server interaction:
- Dropdowns, modals, toggles
- Client-side validation feedback
- Transitions and animations

Use Livewire for anything that touches the server. Don't mix `x-data` with `wire:model` on the same element.

---

## Testing Requirements

```php
// Required: Livewire test for each component
Livewire::test(OrderTable::class)
    ->set('search', 'ORD-001')
    ->assertSee('ORD-001')
    ->assertDontSee('ORD-002');

Livewire::test(CreateOrder::class)
    ->set('form.amount', -5)
    ->call('save')
    ->assertHasErrors(['form.amount' => 'min']);

// Authorization test
Livewire::actingAs($unauthorizedUser)
    ->test(CreateOrder::class)
    ->call('save')
    ->assertForbidden();
```

---

## Deployment Gates

- [ ] All action methods have `$this->authorize()` or equivalent
- [ ] All public properties that shouldn't change are `#[Locked]`
- [ ] No sensitive data in public properties
- [ ] `wire:model.lazy` or `.blur` on all text inputs
- [ ] Livewire full-page components have `<Title>` set
- [ ] Telescope/Debugbar disabled in production
- [ ] `php artisan livewire:publish` assets cached
