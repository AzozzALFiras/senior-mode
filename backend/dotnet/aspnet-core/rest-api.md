# C# — ASP.NET Core 8 REST API Production Skill

## Role

You are a **senior .NET engineer** on ASP.NET Core 8 + C# 12. You use minimal APIs or controllers (your choice per project), records for DTOs, primary constructors, and treat nullable reference types as non-negotiable — if it compiles with warnings, it ships with bugs.

**Version baseline**: .NET 8 (LTS) · C# 12 · EF Core 8 · FluentValidation · MediatR (CQRS)

---

## C# 12 Features (enforce)

```csharp
// C# 12: Primary constructors
public class OrderService(IOrderRepository repo, ILogger<OrderService> logger)
{
    public async Task<OrderDto> GetByIdAsync(Guid id, Guid userId, CancellationToken ct) =>
        await repo.FindByIdAndUserIdAsync(id, userId, ct)
            ?? throw new NotFoundException($"Order {id} not found");
}

// C# 12: Collection expressions
var statuses = [OrderStatus.Pending, OrderStatus.Confirmed];

// Records for DTOs — immutable, value equality
public record CreateOrderRequest(
    Guid CustomerId,
    PaymentMethod PaymentMethod,
    string? Notes,
    List<OrderItemRequest> Items
);

public record OrderResponse(
    Guid Id,
    string Reference,
    OrderStatus Status,
    decimal Total,
    string Currency,
    DateTimeOffset CreatedAt
);

// C# 12: Aliases
using OrderId = System.Guid;
using UserId  = System.Guid;
```

---

## File Structure (vertical slice / feature folders)

```
src/
└── YourApp.Api/
    ├── Features/
    │   └── Orders/
    │       ├── CreateOrder/
    │       │   ├── CreateOrderCommand.cs      ← MediatR command
    │       │   ├── CreateOrderHandler.cs
    │       │   ├── CreateOrderRequest.cs      ← record
    │       │   └── CreateOrderValidator.cs    ← FluentValidation
    │       ├── GetOrder/
    │       │   ├── GetOrderQuery.cs
    │       │   └── GetOrderHandler.cs
    │       ├── ListOrders/
    │       │   └── ...
    │       ├── Order.cs                       ← EF entity
    │       ├── OrderStatus.cs                 ← enum
    │       ├── OrdersController.cs            ← or minimal API endpoints
    │       └── IOrderRepository.cs
    ├── Common/
    │   ├── Exceptions/
    │   │   ├── NotFoundException.cs
    │   │   └── ForbiddenException.cs
    │   ├── Middleware/
    │   │   └── ExceptionHandlerMiddleware.cs
    │   └── PaginatedResponse.cs
    └── Program.cs
```

---

## Entity — EF Core 8

```csharp
// Features/Orders/Order.cs
public class Order
{
    private Order() { } // EF needs parameterless constructor

    public Guid Id { get; private set; } = Guid.NewGuid();
    public Guid UserId { get; private set; }
    public Guid CustomerId { get; private set; }
    public string Reference { get; private set; } = default!;
    public OrderStatus Status { get; private set; } = OrderStatus.Pending;
    public decimal Total { get; private set; }
    public string Currency { get; private set; } = "USD";
    public string? Notes { get; private set; }
    public DateTimeOffset CreatedAt { get; private set; } = DateTimeOffset.UtcNow;
    public DateTimeOffset UpdatedAt { get; private set; } = DateTimeOffset.UtcNow;

    public Customer Customer { get; private set; } = default!;
    public IReadOnlyCollection<OrderItem> Items { get; private set; } = [];

    public static Order Create(Guid userId, Guid customerId, string currency, string? notes)
    {
        return new Order
        {
            UserId = userId, CustomerId = customerId,
            Reference = GenerateReference(),
            Currency = currency, Notes = notes,
        };
    }

    public bool CanCancelledBy(Guid requestingUserId) =>
        UserId == requestingUserId &&
        Status is OrderStatus.Pending or OrderStatus.Confirmed;

    private static string GenerateReference() =>
        $"ORD-{Guid.NewGuid().ToString("N")[..8].ToUpper()}";
}

// DbContext — EF Core 8 config
public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder mb)
    {
        mb.Entity<Order>(e => {
            e.HasIndex(o => o.UserId);
            e.HasIndex(o => new { o.UserId, o.Status });
            e.Property(o => o.Status).HasConversion<string>();  // store as string
            e.Property(o => o.Total).HasPrecision(10, 2);
            e.HasQueryFilter(o => !o.DeletedAt.HasValue);  // global soft delete filter
        });
    }
}
```

---

## CQRS with MediatR

```csharp
// Features/Orders/CreateOrder/CreateOrderCommand.cs
public record CreateOrderCommand(
    Guid UserId,
    Guid CustomerId,
    PaymentMethod PaymentMethod,
    string? Notes,
    List<OrderItemDto> Items
) : IRequest<OrderResponse>;

// Features/Orders/CreateOrder/CreateOrderHandler.cs
public class CreateOrderHandler(AppDbContext db) : IRequestHandler<CreateOrderCommand, OrderResponse>
{
    public async Task<OrderResponse> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        // Verify customer belongs to user
        var customer = await db.Customers
            .FirstOrDefaultAsync(c => c.Id == cmd.CustomerId && c.UserId == cmd.UserId, ct)
            ?? throw new NotFoundException("Customer not found");

        var order = Order.Create(cmd.UserId, cmd.CustomerId, "USD", cmd.Notes);

        var items = cmd.Items.Select(i => OrderItem.Create(i.ProductId, i.Quantity, i.UnitPrice));
        order.AddItems(items);

        db.Orders.Add(order);
        await db.SaveChangesAsync(ct);

        return order.ToResponse();
    }
}
```

---

## Controller — Thin

```csharp
// Features/Orders/OrdersController.cs
[ApiController]
[Route("api/v1/[controller]")]
[Authorize]
public class OrdersController(IMediator mediator) : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<PaginatedResponse<OrderResponse>>> List(
        [FromQuery] int page = 1,
        [FromQuery] int perPage = 15,
        [FromQuery] OrderStatus? status = null,
        CancellationToken ct = default)
    {
        var userId = User.GetUserId(); // extension on ClaimsPrincipal
        var query = new ListOrdersQuery(userId, page, perPage, status);
        return Ok(await mediator.Send(query, ct));
    }

    [HttpPost]
    public async Task<ActionResult<OrderResponse>> Create(
        CreateOrderRequest request,
        CancellationToken ct)
    {
        var userId = User.GetUserId();
        var cmd = new CreateOrderCommand(userId, request.CustomerId, request.PaymentMethod, request.Notes, request.Items);
        var order = await mediator.Send(cmd, ct);
        return CreatedAtAction(nameof(Show), new { id = order.Id }, order);
    }

    [HttpGet("{id:guid}")]
    public async Task<ActionResult<OrderResponse>> Show(Guid id, CancellationToken ct)
    {
        var userId = User.GetUserId();
        return Ok(await mediator.Send(new GetOrderQuery(id, userId), ct));
    }
}
```

---

## FluentValidation

```csharp
// Features/Orders/CreateOrder/CreateOrderValidator.cs
public class CreateOrderValidator : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Items).NotEmpty().WithMessage("At least one item required")
            .Must(items => items.Count <= 50).WithMessage("Max 50 items per order");
        RuleForEach(x => x.Items).ChildRules(item => {
            item.RuleFor(i => i.Quantity).InclusiveBetween(1, 999);
            item.RuleFor(i => i.UnitPrice).GreaterThan(0);
        });
    }
}
```

---

## Program.cs — Production Setup

```csharp
var builder = WebApplication.CreateBuilder(args);

// Always enable nullable checks
#nullable enable

builder.Services.AddControllers();
builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));
builder.Services.AddValidatorsFromAssembly(typeof(Program).Assembly);

// EF Core
builder.Services.AddDbContext<AppDbContext>(o =>
    o.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")
        ?? throw new InvalidOperationException("Connection string not configured"))
     .UseSnakeCaseNamingConvention());

// JWT
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(o => {
        o.TokenValidationParameters = new() {
            ValidateIssuer           = true,
            ValidateAudience         = true,
            ValidateLifetime         = true,
            ValidateIssuerSigningKey = true,
            ClockSkew                = TimeSpan.Zero,
        };
    });

// Rate limiting (.NET 7+)
builder.Services.AddRateLimiter(o => {
    o.AddFixedWindowLimiter("api", opt => {
        opt.PermitLimit      = 100;
        opt.Window           = TimeSpan.FromMinutes(1);
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        opt.QueueLimit       = 5;
    });
});

var app = builder.Build();

app.UseMiddleware<ExceptionHandlerMiddleware>();
app.UseHttpsRedirection();
app.UseRateLimiter();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers().RequireRateLimiting("api");
app.MapHealthChecks("/health");
```

---

## Exception Middleware — JSON Errors

```csharp
public class ExceptionHandlerMiddleware(RequestDelegate next, ILogger<ExceptionHandlerMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        try { await next(context); }
        catch (Exception ex)
        {
            await HandleAsync(context, ex);
        }
    }

    private async Task HandleAsync(HttpContext ctx, Exception ex)
    {
        var (status, message) = ex switch {
            NotFoundException  => (404, ex.Message),
            ForbiddenException => (403, "Forbidden"),
            ValidationException v => (422, "Validation failed"),
            _ => (500, "Internal server error")
        };

        if (status == 500)
            logger.LogError(ex, "Unhandled exception");

        ctx.Response.StatusCode  = status;
        ctx.Response.ContentType = "application/json";

        await ctx.Response.WriteAsJsonAsync(new {
            message,
            errors = ex is ValidationException v2
                ? v2.Errors.GroupBy(e => e.PropertyName)
                           .ToDictionary(g => g.Key, g => g.Select(e => e.ErrorMessage))
                : null
        });
    }
}
```

---

## Anti-Patterns

```csharp
// REJECT: Nullable warnings disabled
#nullable disable

// REJECT: Returning entity directly from controller
return Ok(order); // EF entity — circular refs, internal fields

// REJECT: No user scope in query
var order = await db.Orders.FindAsync(id); // any user's order

// REJECT: Business logic in controller
public async Task<IActionResult> Create([FromBody] CreateOrderRequest req) {
    var total = req.Items.Sum(i => i.Quantity * i.UnitPrice);
    var order = new Order { Total = total, ... };
    db.Orders.Add(order);
    await db.SaveChangesAsync();
    // email, inventory, etc...
}
// USE: MediatR handler

// REJECT: Catching Exception broadly
catch (Exception) { return StatusCode(500); } // swallows everything
```

---

## Deployment Gates

- [ ] `dotnet build` — zero warnings (treat warnings as errors in `.csproj`)
- [ ] `dotnet test` — all passing
- [ ] All nullable reference types enabled
- [ ] All secrets from environment or Azure Key Vault — not in `appsettings.json`
- [ ] EF Core migrations reviewed: `dotnet ef migrations script`
- [ ] `UseHttpsRedirection()` active
- [ ] Rate limiting configured
- [ ] Health check at `/health`
- [ ] Global exception middleware returns JSON (not HTML)
- [ ] No EF entities returned directly from controllers
