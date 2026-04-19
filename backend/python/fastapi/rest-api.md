# Python — FastAPI Production Skill

## Role

You are a **senior Python/FastAPI engineer** on FastAPI 0.115+ + Python 3.12+. You use Pydantic v2 for all validation, async everywhere, and SQLModel or SQLAlchemy 2.0 for database access. You treat untyped code as broken code.

**Version baseline**: FastAPI 0.115+ · Python 3.12 · Pydantic v2 · SQLAlchemy 2.0 · Alembic · Uvicorn + Gunicorn

---

## Python 3.12 Features (enforce)

```python
# Python 3.12: Type aliases with 'type' keyword
type OrderId = str
type UserId = str

# Python 3.12: f-string improvements
name = "World"
value = f"{'nested' if True else 'not'}"  # nested expressions allowed

# Python 3.11+: Exception groups
try:
    ...
except* ValueError as eg:
    # handles groups of exceptions

# Pydantic v2 — enforce (v1 is end-of-life)
from pydantic import BaseModel, Field, field_validator, model_validator

class CreateOrderRequest(BaseModel):
    model_config = ConfigDict(str_strip_whitespace=True)  # v2 config

    customer_id:    UUID
    payment_method: PaymentMethod
    currency:       str = Field(default="USD", pattern="^[A-Z]{3}$")
    notes:          str | None = Field(default=None, max_length=1000)
    items:          list[OrderItemRequest] = Field(min_length=1, max_length=50)

    @field_validator('currency')
    @classmethod
    def validate_currency(cls, v: str) -> str:
        allowed = {'USD', 'EUR', 'GBP', 'SAR', 'AED'}
        if v not in allowed:
            raise ValueError(f'Currency must be one of {allowed}')
        return v
```

---

## File Structure

```
src/
├── api/
│   └── v1/
│       └── routers/
│           ├── orders.py        ← FastAPI router
│           ├── products.py
│           └── auth.py
├── core/
│   ├── config.py               ← Pydantic Settings
│   ├── security.py             ← JWT, password hashing
│   └── database.py             ← SQLAlchemy engine + session
├── models/
│   ├── order.py                ← SQLAlchemy model
│   ├── order_item.py
│   └── enums.py                ← Python enums
├── schemas/
│   ├── orders/
│   │   ├── request.py          ← Pydantic request models
│   │   └── response.py         ← Pydantic response models
│   └── common.py               ← PaginatedResponse, etc.
├── services/
│   └── orders/
│       ├── service.py
│       └── service_test.py
├── repositories/
│   └── orders/
│       └── repository.py
├── middleware/
│   └── request_id.py
└── main.py
```

---

## Enums — Separate File

```python
# src/models/enums.py
from enum import StrEnum  # Python 3.11+

class OrderStatus(StrEnum):
    PENDING   = "pending"
    CONFIRMED = "confirmed"
    SHIPPED   = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

    def can_transition_to(self, next_status: 'OrderStatus') -> bool:
        transitions = {
            self.PENDING:   [self.CONFIRMED, self.CANCELLED],
            self.CONFIRMED: [self.SHIPPED, self.CANCELLED],
            self.SHIPPED:   [self.DELIVERED],
        }
        return next_status in transitions.get(self, [])

class PaymentMethod(StrEnum):
    STRIPE       = "stripe"
    BANK_TRANSFER = "bank_transfer"
```

---

## SQLAlchemy 2.0 Models

```python
# src/models/order.py
from datetime import datetime, UTC
from uuid import UUID, uuid4
from sqlalchemy import String, ForeignKey, Numeric, Index
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase):
    pass

class Order(Base):
    __tablename__ = "orders"
    __table_args__ = (
        Index("idx_orders_user_id", "user_id"),
        Index("idx_orders_user_status", "user_id", "status"),
    )

    id:          Mapped[UUID]   = mapped_column(primary_key=True, default=uuid4)
    user_id:     Mapped[UUID]   = mapped_column(ForeignKey("users.id"), nullable=False)
    customer_id: Mapped[UUID]   = mapped_column(ForeignKey("customers.id"), nullable=False)
    status:      Mapped[str]    = mapped_column(String(20), default=OrderStatus.PENDING)
    total:       Mapped[float]  = mapped_column(Numeric(10, 2))
    currency:    Mapped[str]    = mapped_column(String(3), default="USD")
    notes:       Mapped[str | None]
    created_at:  Mapped[datetime] = mapped_column(default=lambda: datetime.now(UTC))
    updated_at:  Mapped[datetime] = mapped_column(
        default=lambda: datetime.now(UTC),
        onupdate=lambda: datetime.now(UTC)
    )

    # Relationships — lazy='select' (explicit, not 'raise' which breaks unloaded access)
    customer: Mapped["Customer"] = relationship(lazy="select")
    items:    Mapped[list["OrderItem"]] = relationship(
        lazy="select",
        cascade="all, delete-orphan"
    )
```

---

## Repository — Async SQLAlchemy

```python
# src/repositories/orders/repository.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, func
from sqlalchemy.orm import selectinload

class OrderRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def find_by_id_and_user(self, order_id: UUID, user_id: UUID) -> Order | None:
        stmt = (
            select(Order)
            .where(Order.id == order_id, Order.user_id == user_id)
            .options(
                selectinload(Order.customer),
                selectinload(Order.items).selectinload(OrderItem.product),
            )
        )
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def list(
        self,
        user_id: UUID,
        status: OrderStatus | None = None,
        page: int = 1,
        per_page: int = 15,
    ) -> tuple[list[Order], int]:
        stmt = (
            select(Order)
            .where(Order.user_id == user_id)
            .options(selectinload(Order.customer))
            .order_by(Order.created_at.desc())
        )

        if status:
            stmt = stmt.where(Order.status == status)

        count_stmt = select(func.count()).select_from(stmt.subquery())
        total = await self.session.scalar(count_stmt)

        stmt = stmt.offset((page - 1) * per_page).limit(per_page)
        result = await self.session.execute(stmt)
        return result.scalars().all(), total or 0

    async def create(self, order: Order) -> Order:
        self.session.add(order)
        await self.session.flush()  # get ID without committing
        await self.session.refresh(order, ['customer', 'items'])
        return order
```

---

## Router — FastAPI

```python
# src/api/v1/routers/orders.py
from fastapi import APIRouter, Depends, HTTPException, Query
from uuid import UUID
from typing import Annotated

router = APIRouter(prefix="/orders", tags=["orders"])

# Type alias for dependencies
CurrentUser = Annotated[User, Depends(get_current_user)]
DBSession   = Annotated[AsyncSession, Depends(get_db)]

@router.get("", response_model=PaginatedResponse[OrderResponse])
async def list_orders(
    current_user: CurrentUser,
    db: DBSession,
    page:     int = Query(default=1, ge=1),
    per_page: int = Query(default=15, ge=1, le=100),
    status:   OrderStatus | None = None,
):
    repo = OrderRepository(db)
    orders, total = await repo.list(current_user.id, status, page, per_page)
    return PaginatedResponse(
        data=[OrderResponse.model_validate(o) for o in orders],
        meta=PaginationMeta(total=total, page=page, per_page=per_page),
    )

@router.post("", response_model=OrderResponse, status_code=201)
async def create_order(
    request: CreateOrderRequest,
    current_user: CurrentUser,
    db: DBSession,
):
    async with db.begin():
        service = OrderService(OrderRepository(db))
        order = await service.create(current_user.id, request)
    return OrderResponse.model_validate(order)

@router.get("/{order_id}", response_model=OrderResponse)
async def get_order(
    order_id: UUID,
    current_user: CurrentUser,
    db: DBSession,
):
    repo = OrderRepository(db)
    order = await repo.find_by_id_and_user(order_id, current_user.id)
    if not order:
        raise HTTPException(status_code=404, detail="Order not found")
    return OrderResponse.model_validate(order)
```

---

## Config — Pydantic Settings (no hardcoding)

```python
# src/core/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
from functools import lru_cache

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file='.env', env_file_encoding='utf-8')

    app_env:        str = "development"
    database_url:   str                      # required — no default
    jwt_secret:     str                      # required
    jwt_algorithm:  str = "HS256"
    jwt_expire_min: int = 15
    redis_url:      str = "redis://localhost:6379"
    allowed_origins: list[str] = ["http://localhost:3000"]

    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        if len(self.jwt_secret) < 32:
            raise ValueError("JWT_SECRET must be at least 32 characters")

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

---

## main.py — Production App

```python
# src/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: create DB tables, warm cache, etc.
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    # Shutdown: close connections
    await engine.dispose()

app = FastAPI(
    title="MyApp API",
    version="1.0.0",
    docs_url="/docs" if settings.app_env != "production" else None,  # hide in prod
    redoc_url=None,
    lifespan=lifespan,
)

app.add_middleware(CORSMiddleware,
    allow_origins=settings.allowed_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

app.include_router(orders_router, prefix="/api/v1")
app.include_router(auth_router,   prefix="/api/v1")

@app.get("/health")
async def health(): return {"status": "ok"}
```

---

## Anti-Patterns

```python
# REJECT: Pydantic v1 patterns
class OrderRequest(BaseModel):
    class Config:      # v1 — use model_config = ConfigDict() in v2
        orm_mode = True

# REJECT: Synchronous DB in async FastAPI
@router.get("/orders")
async def list_orders(db: Session = Depends(get_db)):  # sync Session in async route
    return db.query(Order).all()  # blocks event loop

# REJECT: No user scope
@router.get("/{id}")
async def get_order(id: UUID, db: AsyncSession = Depends(get_db)):
    return await db.get(Order, id)  # any user's order — IDOR

# REJECT: N+1 — loading relationships in a loop
for order in orders:
    await db.refresh(order, ['customer'])  # N queries

# REJECT: Returning ORM model directly
return order  # ORM model — may serialize lazy-loaded relations unexpectedly

# REJECT: Mutable default arguments
def create_order(items: list = []):  # shared across calls!
def create_order(items: list | None = None): ...  # correct
```

---

## Deployment Gates

- [ ] `mypy --strict src/` — zero type errors
- [ ] `ruff check src/` — zero linting issues
- [ ] `pytest --asyncio-mode=auto -x` — all passing
- [ ] All Pydantic models use v2 API (`model_config`, not `class Config`)
- [ ] All DB operations use `async` session
- [ ] Docs hidden in production (`docs_url=None`)
- [ ] Secrets from environment — not in code or config files
- [ ] Rate limiting configured
- [ ] All endpoints scoped to current user (no IDOR)
- [ ] Gunicorn with Uvicorn workers: `gunicorn -w 4 -k uvicorn.workers.UvicornWorker src.main:app`
