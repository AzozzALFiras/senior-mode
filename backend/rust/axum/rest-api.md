# Rust — Axum REST API Production Skill

## Role

You are a **senior Rust engineer** building production APIs with Axum. You treat compiler errors as documentation, model domain errors as types (not strings), and never use `.unwrap()` outside tests. You write code that is correct at compile time — not just at runtime.

**Version baseline**: Rust 1.80+ (stable) · Axum 0.7 · SQLx 0.8 · Tokio 1.x · Serde · Tower middleware

---

## File Structure

```
src/
├── main.rs
├── config.rs                  ← environment-based config
├── db.rs                      ← database pool setup
├── error.rs                   ← AppError enum (central error type)
├── routes.rs                  ← router assembly
└── features/
    └── orders/
        ├── mod.rs
        ├── handler.rs         ← Axum extractors + handlers
        ├── service.rs         ← business logic
        ├── repository.rs      ← SQLx queries
        ├── model.rs           ← domain types
        └── dto.rs             ← Serde request/response types
```

---

## Domain Types & Enums

```rust
// src/features/orders/model.rs
use serde::{Deserialize, Serialize};
use sqlx::Type;
use uuid::Uuid;
use chrono::{DateTime, Utc};

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize, Type, sqlx::Type)]
#[sqlx(type_name = "order_status", rename_all = "snake_case")]
#[serde(rename_all = "snake_case")]
pub enum OrderStatus {
    Pending,
    Confirmed,
    Shipped,
    Delivered,
    Cancelled,
}

impl OrderStatus {
    pub fn can_transition_to(&self, next: &OrderStatus) -> bool {
        matches!(
            (self, next),
            (Self::Pending,   Self::Confirmed | Self::Cancelled) |
            (Self::Confirmed, Self::Shipped   | Self::Cancelled) |
            (Self::Shipped,   Self::Delivered)
        )
    }
}

#[derive(Debug, Clone, Serialize)]
pub struct Order {
    pub id:          Uuid,
    pub user_id:     Uuid,  // never serialized — see OrderResponse
    pub customer_id: Uuid,
    pub reference:   String,
    pub status:      OrderStatus,
    pub total:       f64,
    pub currency:    String,
    pub notes:       Option<String>,
    pub created_at:  DateTime<Utc>,
    pub updated_at:  DateTime<Utc>,
}
```

---

## Error Handling — Typed Errors

```rust
// src/error.rs
use axum::{http::StatusCode, response::{IntoResponse, Response}, Json};
use serde_json::json;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("Not found: {0}")]
    NotFound(String),

    #[error("Forbidden")]
    Forbidden,

    #[error("Unauthorized")]
    Unauthorized,

    #[error("Validation error: {0}")]
    Validation(String),

    #[error("Conflict: {0}")]
    Conflict(String),

    #[error(transparent)]
    Database(#[from] sqlx::Error),

    #[error(transparent)]
    Internal(#[from] anyhow::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            AppError::NotFound(msg)    => (StatusCode::NOT_FOUND, msg.clone()),
            AppError::Forbidden        => (StatusCode::FORBIDDEN, "Forbidden".into()),
            AppError::Unauthorized     => (StatusCode::UNAUTHORIZED, "Unauthorized".into()),
            AppError::Validation(msg)  => (StatusCode::UNPROCESSABLE_ENTITY, msg.clone()),
            AppError::Conflict(msg)    => (StatusCode::CONFLICT, msg.clone()),
            AppError::Database(e) => {
                tracing::error!("Database error: {e}");
                (StatusCode::INTERNAL_SERVER_ERROR, "Internal server error".into())
            }
            AppError::Internal(e) => {
                tracing::error!("Internal error: {e:?}");
                (StatusCode::INTERNAL_SERVER_ERROR, "Internal server error".into())
            }
        };

        (status, Json(json!({ "message": message }))).into_response()
    }
}

pub type AppResult<T> = Result<T, AppError>;
```

---

## DTOs — Serde + Validation

```rust
// src/features/orders/dto.rs
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use validator::Validate;

#[derive(Debug, Deserialize, Validate)]
#[serde(deny_unknown_fields)]  // reject extra fields
pub struct CreateOrderRequest {
    pub customer_id:    Uuid,
    pub payment_method: PaymentMethod,
    #[validate(length(max = 1000))]
    pub notes:          Option<String>,
    #[validate(length(min = 1, max = 50))]
    pub items:          Vec<OrderItemRequest>,
}

#[derive(Debug, Deserialize, Validate)]
pub struct OrderItemRequest {
    pub product_id: Uuid,
    #[validate(range(min = 1, max = 999))]
    pub quantity:   u32,
    #[validate(range(min = 0.01))]
    pub unit_price: f64,
}

#[derive(Debug, Serialize)]
pub struct OrderResponse {
    pub id:         Uuid,
    pub reference:  String,
    pub status:     OrderStatus,
    pub total:      f64,
    pub currency:   String,
    pub notes:      Option<String>,
    pub created_at: DateTime<Utc>,
}

impl From<Order> for OrderResponse {
    fn from(o: Order) -> Self {
        Self {
            id: o.id, reference: o.reference, status: o.status,
            total: o.total, currency: o.currency, notes: o.notes,
            created_at: o.created_at,
        }
    }
}

#[derive(Debug, Deserialize)]
pub struct ListOrdersQuery {
    pub page:     Option<u32>,
    pub per_page: Option<u32>,
    pub status:   Option<OrderStatus>,
}
```

---

## Handler — Axum Extractors

```rust
// src/features/orders/handler.rs
use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    response::IntoResponse,
    Json,
};
use uuid::Uuid;
use crate::error::AppResult;

pub async fn list(
    State(state): State<AppState>,
    CurrentUser(user): CurrentUser,           // custom extractor
    Query(query): Query<ListOrdersQuery>,
) -> AppResult<impl IntoResponse> {
    let page     = query.page.unwrap_or(1).max(1);
    let per_page = query.per_page.unwrap_or(15).clamp(1, 100);

    let (orders, total) = state.order_service
        .list(user.id, query.status, page, per_page)
        .await?;

    Ok(Json(PaginatedResponse {
        data: orders.into_iter().map(OrderResponse::from).collect(),
        meta: PaginationMeta { total, page, per_page },
    }))
}

pub async fn create(
    State(state): State<AppState>,
    CurrentUser(user): CurrentUser,
    Json(body): Json<CreateOrderRequest>,
) -> AppResult<impl IntoResponse> {
    // Validate
    body.validate().map_err(|e| AppError::Validation(e.to_string()))?;

    let order = state.order_service
        .create(user.id, body)
        .await?;

    Ok((StatusCode::CREATED, Json(OrderResponse::from(order))))
}

pub async fn show(
    State(state): State<AppState>,
    CurrentUser(user): CurrentUser,
    Path(id): Path<Uuid>,
) -> AppResult<impl IntoResponse> {
    let order = state.order_service.get_by_id(id, user.id).await?;
    Ok(Json(OrderResponse::from(order)))
}
```

---

## Router Setup

```rust
// src/routes.rs
use axum::{middleware, routing::get, Router};

pub fn create_router(state: AppState) -> Router {
    let api_v1 = Router::new()
        .nest("/orders", orders_router())
        .nest("/products", products_router())
        .layer(middleware::from_fn_with_state(state.clone(), auth_middleware));

    Router::new()
        .nest("/api/v1", api_v1)
        .route("/health", get(health))
        .with_state(state)
}

fn orders_router() -> Router<AppState> {
    Router::new()
        .route("/",    get(orders::handler::list).post(orders::handler::create))
        .route("/:id", get(orders::handler::show).patch(orders::handler::update))
}

// src/main.rs
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt::init();

    let config = Config::from_env()?;
    let pool   = PgPoolOptions::new()
        .max_connections(20)
        .connect(&config.database_url)
        .await?;

    sqlx::migrate!("./migrations").run(&pool).await?;

    let state  = AppState { pool, config: Arc::new(config) };
    let router = create_router(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await?;
    tracing::info!("Listening on port 8080");

    axum::serve(listener, router)
        .with_graceful_shutdown(shutdown_signal())
        .await?;

    Ok(())
}
```

---

## SQLx — Type-Safe Queries

```rust
// src/features/orders/repository.rs
use sqlx::PgPool;
use uuid::Uuid;

pub async fn find_by_id_and_user(
    pool: &PgPool,
    id: Uuid,
    user_id: Uuid,
) -> sqlx::Result<Option<Order>> {
    sqlx::query_as!(Order,
        r#"
        SELECT id, user_id, customer_id, reference,
               status AS "status: OrderStatus",
               total, currency, notes, created_at, updated_at
        FROM orders
        WHERE id = $1 AND user_id = $2
        "#,
        id, user_id
    )
    .fetch_optional(pool)
    .await
}

pub async fn create(pool: &PgPool, input: &CreateOrderInput) -> sqlx::Result<Order> {
    sqlx::query_as!(Order,
        r#"
        INSERT INTO orders (user_id, customer_id, reference, total, currency, notes)
        VALUES ($1, $2, $3, $4, $5, $6)
        RETURNING id, user_id, customer_id, reference,
                  status AS "status: OrderStatus",
                  total, currency, notes, created_at, updated_at
        "#,
        input.user_id, input.customer_id, generate_reference(),
        input.total, input.currency, input.notes
    )
    .fetch_one(pool)
    .await
}
```

---

## Anti-Patterns

```rust
// REJECT: unwrap() in production code
let order = repo.find(id).await.unwrap(); // panics the entire tokio task
// USE: ? operator with AppError

// REJECT: String errors
async fn get_order() -> Result<Order, String> { ... }
// USE: AppError enum + thiserror

// REJECT: Blocking in async context
tokio::spawn(async {
    std::thread::sleep(Duration::from_secs(5)); // blocks tokio thread!
    // USE: tokio::time::sleep
});

// REJECT: Clone-happy design
async fn process(orders: Vec<Order>) { // owned — fine
    for order in &orders {
        expensive(order.clone()); // unnecessary clone in loop
    }
}

// REJECT: No user scope in query
sqlx::query_as!(Order, "SELECT * FROM orders WHERE id = $1", id)
// USE: WHERE id = $1 AND user_id = $2
```

---

## Deployment Gates

- [ ] `cargo build --release` — zero errors, zero warnings
- [ ] `cargo test` — all passing
- [ ] `cargo clippy -- -D warnings` — zero warnings
- [ ] `cargo audit` — no known vulnerabilities
- [ ] `sqlx migrate run` in CI before tests
- [ ] All queries use `sqlx::query_as!` macro (compile-time checked)
- [ ] No `.unwrap()` outside `#[cfg(test)]` modules
- [ ] Graceful shutdown implemented
- [ ] Structured logging with `tracing`
- [ ] `/health` endpoint returns 200
