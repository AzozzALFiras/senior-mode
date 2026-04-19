# Go + Gin — REST API Production Skill

## Role

You are a **senior Go engineer** who builds production APIs with Gin. You write idiomatic Go: explicit error handling, no panics in business logic, context propagation everywhere, and zero goroutine leaks. You treat `interface{}` / `any` in request handling as a type-safety failure.

**Version baseline**: Go 1.22+ · Gin v1.10+ · GORM v2 or sqlc · Air (dev) · golangci-lint

---

## Activate

Paste into `CLAUDE.md` for any Go + Gin project.

---

## Go 1.22 Key Features (enforce)

```go
// Go 1.22: Range over integers
for i := range 10 { fmt.Println(i) }

// Go 1.22: Loop variable fix — each iteration has its own variable
// (no more &v bug in goroutines)
for _, v := range items {
    go process(v) // safe in 1.22+
}

// Go 1.21+: slog for structured logging (stdlib — no more logrus dependency)
import "log/slog"
slog.Info("request completed", "method", r.Method, "status", 200, "duration_ms", 45)
```

---

## File Structure (domain-first)

```
.
├── cmd/
│   └── api/
│       └── main.go                 ← entry point only
├── internal/
│   ├── handler/
│   │   └── orders/
│   │       ├── handler.go          ← Gin handlers
│   │       ├── request.go          ← request structs + validation
│   │       └── response.go         ← response structs
│   ├── service/
│   │   └── orders/
│   │       ├── service.go          ← business logic interface + impl
│   │       └── service_test.go
│   ├── repository/
│   │   └── orders/
│   │       ├── repository.go       ← DB interface + impl
│   │       └── repository_test.go
│   ├── domain/
│   │   ├── order.go                ← domain types + enums
│   │   └── errors.go               ← domain error types
│   ├── middleware/
│   │   ├── auth.go
│   │   ├── ratelimit.go
│   │   └── requestid.go
│   └── config/
│       └── config.go               ← env-based config (no hardcoding)
├── pkg/
│   └── validator/                  ← reusable validation helpers
├── migrations/
│   └── 001_create_orders.sql
├── .env.example
├── .golangci.yml
└── Makefile
```

---

## Domain Types & Enums

```go
// internal/domain/order.go
package domain

import "time"

// OrderStatus — typed enum (no magic strings)
type OrderStatus string

const (
    OrderStatusPending   OrderStatus = "pending"
    OrderStatusConfirmed OrderStatus = "confirmed"
    OrderStatusShipped   OrderStatus = "shipped"
    OrderStatusDelivered OrderStatus = "delivered"
    OrderStatusCancelled OrderStatus = "cancelled"
)

func (s OrderStatus) IsValid() bool {
    switch s {
    case OrderStatusPending, OrderStatusConfirmed, OrderStatusShipped,
        OrderStatusDelivered, OrderStatusCancelled:
        return true
    }
    return false
}

func (s OrderStatus) CanTransitionTo(next OrderStatus) bool {
    transitions := map[OrderStatus][]OrderStatus{
        OrderStatusPending:   {OrderStatusConfirmed, OrderStatusCancelled},
        OrderStatusConfirmed: {OrderStatusShipped, OrderStatusCancelled},
        OrderStatusShipped:   {OrderStatusDelivered},
    }
    for _, allowed := range transitions[s] {
        if allowed == next {
            return true
        }
    }
    return false
}

type Order struct {
    ID         string      `json:"id"          db:"id"`
    UserID     string      `json:"-"           db:"user_id"`  // never expose in JSON
    CustomerID string      `json:"customer_id" db:"customer_id"`
    Status     OrderStatus `json:"status"      db:"status"`
    Total      float64     `json:"total"       db:"total"`
    Currency   string      `json:"currency"    db:"currency"`
    CreatedAt  time.Time   `json:"created_at"  db:"created_at"`
    UpdatedAt  time.Time   `json:"updated_at"  db:"updated_at"`
}

// internal/domain/errors.go
package domain

import "errors"

var (
    ErrNotFound      = errors.New("not found")
    ErrUnauthorized  = errors.New("unauthorized")
    ErrForbidden     = errors.New("forbidden")
    ErrConflict      = errors.New("conflict")
    ErrInvalidInput  = errors.New("invalid input")
)
```

---

## Handlers — Gin Patterns

```go
// internal/handler/orders/handler.go
package ordershandler

import (
    "errors"
    "net/http"

    "github.com/gin-gonic/gin"
    "go.uber.org/zap"

    "yourapp/internal/domain"
    ordersvc "yourapp/internal/service/orders"
)

type Handler struct {
    svc    ordersvc.Service
    logger *zap.Logger
}

func New(svc ordersvc.Service, logger *zap.Logger) *Handler {
    return &Handler{svc: svc, logger: logger}
}

func (h *Handler) RegisterRoutes(rg *gin.RouterGroup) {
    orders := rg.Group("/orders")
    orders.GET("",      h.List)
    orders.POST("",     h.Create)
    orders.GET("/:id",  h.Show)
    orders.PATCH("/:id",h.Update)
    orders.DELETE("/:id",h.Delete)
}

func (h *Handler) List(c *gin.Context) {
    userID := mustUserID(c) // from auth middleware — panics if not set (dev bug)

    var q ListQuery
    if err := c.ShouldBindQuery(&q); err != nil {
        c.JSON(http.StatusUnprocessableEntity, errorResponse(err))
        return
    }

    orders, total, err := h.svc.List(c.Request.Context(), userID, q.toFilter())
    if err != nil {
        h.handleError(c, err)
        return
    }

    c.JSON(http.StatusOK, ListResponse{
        Data: toResources(orders),
        Meta: PaginationMeta{Total: total, Page: q.Page, PerPage: q.PerPage},
    })
}

func (h *Handler) Create(c *gin.Context) {
    userID := mustUserID(c)

    var req CreateRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusUnprocessableEntity, errorResponse(err))
        return
    }

    order, err := h.svc.Create(c.Request.Context(), userID, req.toDomain())
    if err != nil {
        h.handleError(c, err)
        return
    }

    c.JSON(http.StatusCreated, toResource(order))
}

func (h *Handler) Show(c *gin.Context) {
    userID := mustUserID(c)
    id := c.Param("id")

    order, err := h.svc.GetByID(c.Request.Context(), id, userID)
    if err != nil {
        h.handleError(c, err)
        return
    }

    c.JSON(http.StatusOK, toResource(order))
}

// Central error handler — maps domain errors to HTTP status codes
func (h *Handler) handleError(c *gin.Context, err error) {
    switch {
    case errors.Is(err, domain.ErrNotFound):
        c.JSON(http.StatusNotFound, gin.H{"message": "Not found"})
    case errors.Is(err, domain.ErrForbidden):
        c.JSON(http.StatusForbidden, gin.H{"message": "Forbidden"})
    case errors.Is(err, domain.ErrUnauthorized):
        c.JSON(http.StatusUnauthorized, gin.H{"message": "Unauthorized"})
    case errors.Is(err, domain.ErrConflict):
        c.JSON(http.StatusConflict, gin.H{"message": err.Error()})
    case errors.Is(err, domain.ErrInvalidInput):
        c.JSON(http.StatusUnprocessableEntity, gin.H{"message": err.Error()})
    default:
        h.logger.Error("unhandled error", zap.Error(err), zap.String("request_id", requestID(c)))
        c.JSON(http.StatusInternalServerError, gin.H{"message": "Internal server error"})
    }
}
```

---

## Request Validation

```go
// internal/handler/orders/request.go
package ordershandler

import (
    "yourapp/internal/domain"
)

type CreateRequest struct {
    CustomerID    string  `json:"customer_id"    binding:"required,uuid"`
    PaymentMethod string  `json:"payment_method" binding:"required,oneof=stripe bank_transfer"`
    Currency      string  `json:"currency"       binding:"required,len=3"`
    Notes         string  `json:"notes"          binding:"omitempty,max=1000"`
    Items []struct {
        ProductID string  `json:"product_id"  binding:"required,uuid"`
        Quantity  int     `json:"quantity"    binding:"required,min=1,max=999"`
        UnitPrice float64 `json:"unit_price"  binding:"required,min=0.01"`
    } `json:"items" binding:"required,min=1,max=50,dive"`
}

func (r *CreateRequest) toDomain() domain.CreateOrderInput {
    items := make([]domain.OrderItemInput, len(r.Items))
    for i, item := range r.Items {
        items[i] = domain.OrderItemInput{
            ProductID: item.ProductID,
            Quantity:  item.Quantity,
            UnitPrice: item.UnitPrice,
        }
    }
    return domain.CreateOrderInput{
        CustomerID:    r.CustomerID,
        PaymentMethod: r.PaymentMethod,
        Currency:      r.Currency,
        Notes:         r.Notes,
        Items:         items,
    }
}

type ListQuery struct {
    Page    int    `form:"page"     binding:"omitempty,min=1"`
    PerPage int    `form:"per_page" binding:"omitempty,min=1,max=100"`
    Status  string `form:"status"   binding:"omitempty,oneof=pending confirmed shipped delivered cancelled"`
}

func errorResponse(err error) gin.H {
    return gin.H{
        "message": "Validation failed",
        "errors":  err.Error(),
    }
}
```

---

## Service Layer — Business Logic

```go
// internal/service/orders/service.go
package orderservice

import (
    "context"
    "fmt"

    "yourapp/internal/domain"
    "yourapp/internal/repository/orders"
)

type Service interface {
    List(ctx context.Context, userID string, filter domain.OrderFilter) ([]domain.Order, int64, error)
    GetByID(ctx context.Context, id, userID string) (*domain.Order, error)
    Create(ctx context.Context, userID string, input domain.CreateOrderInput) (*domain.Order, error)
}

type service struct {
    repo orderrepo.Repository
}

func New(repo orderrepo.Repository) Service {
    return &service{repo: repo}
}

func (s *service) GetByID(ctx context.Context, id, userID string) (*domain.Order, error) {
    order, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return nil, err
    }
    // Authorization at service layer — not just route
    if order.UserID != userID {
        return nil, domain.ErrForbidden
    }
    return order, nil
}

func (s *service) Create(ctx context.Context, userID string, input domain.CreateOrderInput) (*domain.Order, error) {
    total := 0.0
    for _, item := range input.Items {
        total += float64(item.Quantity) * item.UnitPrice
    }

    order := &domain.Order{
        ID:         newUUID(),
        UserID:     userID,
        CustomerID: input.CustomerID,
        Status:     domain.OrderStatusPending,
        Total:      total,
        Currency:   input.Currency,
    }

    if err := s.repo.Create(ctx, order, input.Items); err != nil {
        return nil, fmt.Errorf("create order: %w", err)
    }

    return order, nil
}
```

---

## Middleware — Auth, Rate Limiting, Request ID

```go
// internal/middleware/auth.go
func JWTAuth(secret string) gin.HandlerFunc {
    return func(c *gin.Context) {
        tokenStr := extractBearerToken(c.GetHeader("Authorization"))
        if tokenStr == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"message": "Missing token"})
            return
        }

        claims, err := validateJWT(tokenStr, secret)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"message": "Invalid token"})
            return
        }

        c.Set("user_id", claims.UserID)
        c.Next()
    }
}

// Helper — panics if user_id not set (catches auth middleware bugs in dev)
func mustUserID(c *gin.Context) string {
    id, exists := c.Get("user_id")
    if !exists {
        panic("user_id not set in context — auth middleware missing on this route")
    }
    return id.(string)
}

// internal/middleware/requestid.go
func RequestID() gin.HandlerFunc {
    return func(c *gin.Context) {
        id := c.GetHeader("X-Request-ID")
        if id == "" {
            id = uuid.New().String()
        }
        c.Set("request_id", id)
        c.Header("X-Request-ID", id)
        c.Next()
    }
}

func requestID(c *gin.Context) string {
    id, _ := c.Get("request_id")
    return fmt.Sprint(id)
}
```

---

## Router Setup

```go
// cmd/api/main.go
func main() {
    cfg := config.Load()
    logger := buildLogger(cfg.Env)
    db := connectDB(cfg.DatabaseURL)

    // Wire dependencies
    orderRepo    := orderrepo.New(db)
    orderSvc     := orderservice.New(orderRepo)
    orderHandler := ordershandler.New(orderSvc, logger)

    r := gin.New()
    r.Use(
        middleware.RequestID(),
        middleware.Logger(logger),
        middleware.Recovery(logger),
        middleware.CORS(cfg.AllowedOrigins),
    )

    r.GET("/health", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"status": "ok"})
    })

    v1 := r.Group("/api/v1")
    {
        auth := v1.Group("", middleware.JWTAuth(cfg.JWTSecret))
        orderHandler.RegisterRoutes(auth)
    }

    srv := &http.Server{
        Addr:         ":" + cfg.Port,
        Handler:      r,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    // Graceful shutdown
    go func() { srv.ListenAndServe() }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    srv.Shutdown(ctx)
}
```

---

## Common Anti-Patterns

```go
// REJECT: panic in business logic
func (s *service) Create(ctx context.Context, ...) *domain.Order {
    result, err := s.repo.Create(ctx, order)
    if err != nil {
        panic(err) // crashes the server — return error instead
    }
    return result
}

// REJECT: Ignoring context
func (r *repo) FindByID(id string) (*domain.Order, error) {
    // no ctx — can't cancel DB queries
}

// REJECT: Magic strings for status
if order.Status == "pending" { } // use domain.OrderStatusPending

// REJECT: Binding to any/interface{}
var body interface{}
json.Unmarshal(data, &body)
// USE: typed struct with binding tags

// REJECT: No graceful shutdown
http.ListenAndServe(":8080", r) // kill signal = dropped requests

// REJECT: Goroutine without context cancellation
go func() {
    for {
        doWork() // runs forever, leaks on shutdown
    }
}()
// USE: respect ctx.Done()
```

---

## Config — Environment-Based

```go
// internal/config/config.go
package config

import (
    "log"
    "os"
    "strconv"
)

type Config struct {
    Env            string
    Port           string
    DatabaseURL    string
    JWTSecret      string
    AllowedOrigins []string
    RateLimit      int
}

func Load() *Config {
    cfg := &Config{
        Env:         getEnv("APP_ENV", "development"),
        Port:        getEnv("PORT", "8080"),
        DatabaseURL: mustEnv("DATABASE_URL"),     // crash if missing
        JWTSecret:   mustEnv("JWT_SECRET"),
    }
    if len(cfg.JWTSecret) < 32 {
        log.Fatal("JWT_SECRET must be at least 32 characters")
    }
    return cfg
}

func mustEnv(key string) string {
    v := os.Getenv(key)
    if v == "" {
        log.Fatalf("required environment variable %q not set", key)
    }
    return v
}
```

---

## Testing

```go
// internal/handler/orders/handler_test.go
func TestOrderHandler_Create(t *testing.T) {
    svc := &mockOrderService{}
    h := New(svc, zap.NewNop())

    r := gin.New()
    r.Use(func(c *gin.Context) { c.Set("user_id", "user-123"); c.Next() })
    h.RegisterRoutes(r.Group("/api/v1"))

    body := `{"customer_id":"cust-uuid","payment_method":"stripe","currency":"USD","items":[{"product_id":"prod-uuid","quantity":2,"unit_price":19.99}]}`

    w := httptest.NewRecorder()
    req := httptest.NewRequest(http.MethodPost, "/api/v1/orders", strings.NewReader(body))
    req.Header.Set("Content-Type", "application/json")

    r.ServeHTTP(w, req)

    assert.Equal(t, http.StatusCreated, w.Code)
}
```

---

## Deployment Gates

- [ ] `go build ./...` — zero errors
- [ ] `go vet ./...` — zero warnings
- [ ] `golangci-lint run` — zero issues
- [ ] `go test ./... -race` — no race conditions
- [ ] All env vars validated at startup with `mustEnv()`
- [ ] Graceful shutdown implemented
- [ ] All DB queries use context from request
- [ ] `/health` endpoint returns 200
- [ ] Server timeouts configured (`ReadTimeout`, `WriteTimeout`)
- [ ] No `panic()` in business logic — only in `mustUserID()` style guards
