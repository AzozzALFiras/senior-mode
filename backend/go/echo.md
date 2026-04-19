# Go + Echo v4 — REST API Production Skill

## Role

You are a **senior Go engineer** using Echo v4 framework. You write idiomatic Go with explicit error handling, context propagation, and typed middleware. You prefer Echo's binder for validation and use `echo.HTTPError` for all API errors.

**Version baseline**: Go 1.22+ · Echo v4.12+ · sqlc or GORM v2 · Zerolog

---

## File Structure

```
.
├── cmd/api/main.go
├── internal/
│   ├── handler/
│   │   └── orders/
│   │       ├── handler.go
│   │       ├── request.go
│   │       └── response.go
│   ├── service/orders/
│   ├── repository/orders/
│   ├── domain/
│   │   ├── order.go
│   │   └── errors.go
│   ├── middleware/
│   │   ├── auth.go
│   │   └── error_handler.go
│   └── config/config.go
└── migrations/
```

---

## Echo Setup

```go
// cmd/api/main.go
func main() {
    cfg := config.Load()

    e := echo.New()
    e.HideBanner = true
    e.Validator = &CustomValidator{validator: validator.New()}
    e.HTTPErrorHandler = middleware.ErrorHandler(logger)

    e.Use(echomiddleware.RequestID())
    e.Use(echomiddleware.RecoverWithConfig(echomiddleware.RecoverConfig{
        LogErrorFunc: func(c echo.Context, err error, stack []byte) error {
            logger.Error().Err(err).Bytes("stack", stack).Msg("panic recovered")
            return nil
        },
    }))
    e.Use(echomiddleware.RateLimiter(echomiddleware.NewRateLimiterMemoryStore(100)))

    e.GET("/health", func(c echo.Context) error {
        return c.JSON(http.StatusOK, echo.Map{"status": "ok"})
    })

    v1 := e.Group("/api/v1", middleware.JWTAuth(cfg.JWTSecret))
    ordershandler.New(orderSvc, logger).RegisterRoutes(v1)

    e.Logger.Fatal(e.Start(":" + cfg.Port))
}

// Custom validator using go-playground/validator
type CustomValidator struct {
    validator *validator.Validate
}

func (cv *CustomValidator) Validate(i any) error {
    if err := cv.validator.Struct(i); err != nil {
        return echo.NewHTTPError(http.StatusUnprocessableEntity, err.Error())
    }
    return nil
}
```

---

## Handler Pattern

```go
// internal/handler/orders/handler.go
type Handler struct {
    svc    orderservice.Service
    logger zerolog.Logger
}

func (h *Handler) RegisterRoutes(g *echo.Group) {
    orders := g.Group("/orders")
    orders.GET("",     h.List)
    orders.POST("",    h.Create)
    orders.GET("/:id", h.Show)
    orders.PATCH("/:id", h.Update)
}

func (h *Handler) Create(c echo.Context) error {
    userID := c.Get("user_id").(string)

    var req CreateRequest
    if err := c.Bind(&req); err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, err.Error())
    }
    if err := c.Validate(&req); err != nil {
        return err
    }

    order, err := h.svc.Create(c.Request().Context(), userID, req.toDomain())
    if err != nil {
        return mapError(err)
    }

    return c.JSON(http.StatusCreated, toResource(order))
}

// Map domain errors to HTTP errors
func mapError(err error) error {
    switch {
    case errors.Is(err, domain.ErrNotFound):
        return echo.NewHTTPError(http.StatusNotFound, "Not found")
    case errors.Is(err, domain.ErrForbidden):
        return echo.NewHTTPError(http.StatusForbidden, "Forbidden")
    case errors.Is(err, domain.ErrConflict):
        return echo.NewHTTPError(http.StatusConflict, err.Error())
    default:
        return echo.NewHTTPError(http.StatusInternalServerError, "Internal server error")
    }
}
```

---

## Global Error Handler

```go
// internal/middleware/error_handler.go
func ErrorHandler(logger zerolog.Logger) echo.HTTPErrorHandler {
    return func(err error, c echo.Context) {
        if c.Response().Committed {
            return
        }

        var he *echo.HTTPError
        if !errors.As(err, &he) {
            logger.Error().Err(err).Str("request_id", c.Response().Header().Get(echo.HeaderXRequestID)).Msg("unhandled error")
            he = &echo.HTTPError{Code: http.StatusInternalServerError, Message: "Internal server error"}
        }

        c.JSON(he.Code, echo.Map{
            "message":    he.Message,
            "request_id": c.Response().Header().Get(echo.HeaderXRequestID),
        })
    }
}
```

---

## Anti-Patterns

```go
// REJECT: Ignoring context
func (r *repo) FindByID(id string) (*domain.Order, error) { ... }
// USE: FindByID(ctx context.Context, id string)

// REJECT: panic in handler
func (h *Handler) Create(c echo.Context) error {
    order := h.svc.Create(...) // assuming no error — panics if nil

// REJECT: Direct string errors
return errors.New("order not found")  // no type checking at call site
// USE: domain.ErrNotFound sentinel errors

// REJECT: No user scope
order, _ := h.svc.GetByID(ctx, c.Param("id"))  // any user's order

// REJECT: Missing validator setup
e := echo.New()
// e.Validator not set — c.Validate() silently does nothing
```

---

## Deployment Gates

- [ ] `go build ./...` passes
- [ ] `golangci-lint run` — zero issues
- [ ] `go test ./... -race` — no races
- [ ] Custom validator registered on Echo instance
- [ ] Global error handler registered — no default HTML errors
- [ ] All handlers propagate request context to service/repo
- [ ] Graceful shutdown with `e.Shutdown(ctx)`
- [ ] `/health` returns 200
