# punq — Examples & Gotchas

> Part of the punq skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Circular Dependencies](#1-circular-dependencies)
  - [2. Registration Order Does Not Matter (for resolution)](#2-registration-order-does-not-matter-for-resolution)
  - [3. Missing Registrations](#3-missing-registrations)
  - [4. Thread Safety](#4-thread-safety)
  - [5. Duplicate Registrations (Last One Wins)](#5-duplicate-registrations-last-one-wins)
  - [6. Unannotated Parameters Are Not Auto-Wired](#6-unannotated-parameters-are-not-auto-wired)
  - [7. Primitive Types and Built-in Types](#7-primitive-types-and-built-in-types)
- [Complete Code Examples](#complete-code-examples)
  - [Example 1: Basic Register and Resolve](#example-1-basic-register-and-resolve)
  - [Example 2: Factory Registration](#example-2-factory-registration)
  - [Example 3: Singleton Pattern](#example-3-singleton-pattern)
  - [Example 4: Real-World Service Layer](#example-4-real-world-service-layer)
  - [Example 5: Testing with Mock Dependencies](#example-5-testing-with-mock-dependencies)
- [Comparison with Alternatives](#comparison-with-alternatives)
- [References](#references)

## Gotchas and Common Mistakes

### 1. Circular Dependencies

punq does **not** detect circular dependencies at registration time. If you create
a circular dependency chain, you will get a `RecursionError` at resolve time.

```python
# THIS WILL FAIL with RecursionError
class A:
    def __init__(self, b: "B"):
        self.b = b

class B:
    def __init__(self, a: A):
        self.a = a

container = punq.Container()
container.register(A)
container.register(B)

# RecursionError: maximum recursion depth exceeded
service = container.resolve(A)
```

**Fix:** Break the cycle by introducing an interface, using a factory, or restructuring
your dependencies. Circular dependencies usually indicate a design problem.

```python
# Fix: break the cycle with a mediator or event system
class EventBus:
    pass

class A:
    def __init__(self, bus: EventBus):
        self.bus = bus

class B:
    def __init__(self, bus: EventBus):
        self.bus = bus
```

### 2. Registration Order Does Not Matter (for resolution)

punq resolves dependencies at `resolve()` time, not at `register()` time. You can
register services in any order. However, all dependencies must be registered before
you call `resolve()`.

```python
import punq

class Repo:
    pass

class Service:
    def __init__(self, repo: Repo):
        self.repo = repo

container = punq.Container()

# Order does not matter:
container.register(Service)
container.register(Repo)

# Both are registered by the time we resolve
service = container.resolve(Service)  # Works fine
```

### 3. Missing Registrations

If you try to resolve a type that has not been registered, or if a dependency of a
registered type is missing, punq raises `punq.MissingDependencyError`.

```python
import punq

class UnregisteredService:
    pass

container = punq.Container()

try:
    container.resolve(UnregisteredService)
except punq.MissingDependencyError as e:
    print(f"Error: {e}")
    # Error: Cannot resolve UnregisteredService
```

**Tip:** Register all services at application startup and resolve early (e.g., in a
health check or startup hook) to catch missing registrations immediately, rather than
discovering them at runtime when a request arrives.

### 4. Thread Safety

punq's `Container` is **not thread-safe** for concurrent registration and resolution.
If you are using punq in a multi-threaded web server:

- **Do all registration at startup** (single-threaded initialization), then
- **Only call `resolve()`** during request handling.

Resolving singletons that are already cached is safe for concurrent reads, but the
first resolution of a singleton in a multi-threaded context could have a race condition.

For thread safety during resolution, consider:

```python
import threading
import punq

class ThreadSafeContainer:
    """Wrapper that adds thread safety to punq.Container."""

    def __init__(self):
        self._container = punq.Container()
        self._lock = threading.Lock()

    def register(self, *args, **kwargs):
        with self._lock:
            self._container.register(*args, **kwargs)

    def resolve(self, service_type):
        with self._lock:
            return self._container.resolve(service_type)
```

### 5. Duplicate Registrations (Last One Wins)

If you register the same service type multiple times, the **last registration wins**.
This is by design and is useful for testing (override with mocks), but can cause
subtle bugs if done accidentally.

```python
import punq

class Logger:
    def __init__(self, level: str):
        self.level = level

container = punq.Container()
container.register(Logger, level="DEBUG")
container.register(Logger, level="ERROR")  # This one wins

logger = container.resolve(Logger)
assert logger.level == "ERROR"
```

### 6. Unannotated Parameters Are Not Auto-Wired

punq relies on type annotations to resolve dependencies. Parameters without type
annotations will not be auto-wired and must be provided explicitly via `register()`.

```python
import punq

class Service:
    def __init__(self, name, timeout):  # No type annotations!
        self.name = name
        self.timeout = timeout

container = punq.Container()

# Must provide values explicitly since punq cannot infer types
container.register(Service, name="my-service", timeout=30)

svc = container.resolve(Service)
assert svc.name == "my-service"
```

### 7. Primitive Types and Built-in Types

Avoid registering built-in types like `str`, `int`, or `dict` directly. They are
too generic and will conflict if multiple services need different string or int values.

```python
import punq

# BAD: Registering primitives globally
container = punq.Container()
container.register(str, instance="hello")  # Every str dependency gets "hello"

# GOOD: Use explicit kwargs or wrapper classes
from dataclasses import dataclass

@dataclass
class AppName:
    value: str

container = punq.Container()
container.register(AppName, instance=AppName(value="MyApp"))
```

---

## Complete Code Examples

### Example 1: Basic Register and Resolve

```python
"""
Basic usage: register types and resolve them with auto-wiring.
"""

import punq

class Logger:
    def log(self, message: str) -> None:
        print(f"[LOG] {message}")

class UserRepository:
    def __init__(self, logger: Logger):
        self.logger = logger

    def get_user(self, user_id: int) -> dict:
        self.logger.log(f"Fetching user {user_id}")
        return {"id": user_id, "name": "Alice", "email": "alice@example.com"}

class UserService:
    def __init__(self, repo: UserRepository, logger: Logger):
        self.repo = repo
        self.logger = logger

    def get_user_profile(self, user_id: int) -> dict:
        self.logger.log(f"Building profile for user {user_id}")
        user = self.repo.get_user(user_id)
        return {"display_name": user["name"], "contact": user["email"]}

# Setup
container = punq.Container()
container.register(Logger)
container.register(UserRepository)
container.register(UserService)

# Usage
service = container.resolve(UserService)
profile = service.get_user_profile(42)
print(profile)
# Output:
# [LOG] Building profile for user 42
# [LOG] Fetching user 42
# {'display_name': 'Alice', 'contact': 'alice@example.com'}
```

### Example 2: Factory Registration

```python
"""
Using factory functions for complex construction logic.
"""

import punq
from dataclasses import dataclass

@dataclass
class DatabaseConfig:
    host: str
    port: int
    database: str

class DatabasePool:
    def __init__(self, dsn: str, pool_size: int):
        self.dsn = dsn
        self.pool_size = pool_size
        print(f"Created pool: {dsn} (size={pool_size})")

    def query(self, sql: str) -> list:
        return [{"result": "data"}]

def create_db_pool(config: DatabaseConfig) -> DatabasePool:
    """Factory that constructs a DatabasePool from configuration."""
    dsn = f"postgresql://{config.host}:{config.port}/{config.database}"
    return DatabasePool(dsn=dsn, pool_size=10)

# Setup
container = punq.Container()
container.register(
    DatabaseConfig,
    instance=DatabaseConfig(host="localhost", port=5432, database="myapp"),
)
container.register(DatabasePool, factory=create_db_pool, scope=punq.Scope.singleton)

# Resolve -- factory is called with injected DatabaseConfig
pool = container.resolve(DatabasePool)
print(pool.dsn)       # postgresql://localhost:5432/myapp
print(pool.pool_size)  # 10

# Singleton: same pool instance returned
pool2 = container.resolve(DatabasePool)
assert pool is pool2
```

### Example 3: Singleton Pattern

```python
"""
Singleton services are created once and shared everywhere.
"""

import punq

class ConfigManager:
    """Application configuration -- should be a singleton."""

    _load_count = 0

    def __init__(self):
        ConfigManager._load_count += 1
        self.settings = {"debug": True, "log_level": "INFO"}
        print(f"ConfigManager loaded (count: {self._load_count})")

    def get(self, key: str, default=None):
        return self.settings.get(key, default)

class ServiceA:
    def __init__(self, config: ConfigManager):
        self.config = config

class ServiceB:
    def __init__(self, config: ConfigManager):
        self.config = config

# Setup with singleton config
container = punq.Container()
container.register(ConfigManager, scope=punq.Scope.singleton)
container.register(ServiceA)
container.register(ServiceB)

a = container.resolve(ServiceA)
b = container.resolve(ServiceB)

# Both services share the same ConfigManager instance
assert a.config is b.config
print(f"Same config? {a.config is b.config}")  # True
# ConfigManager was only instantiated once
assert ConfigManager._load_count == 1
```

### Example 4: Real-World Service Layer

```python
"""
A realistic service layer for an e-commerce application using punq
for dependency injection with interfaces, implementations, and layered architecture.
"""

import abc
import punq
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Optional
from uuid import uuid4

# --- Value Objects ---

@dataclass(frozen=True)
class Money:
    amount: float
    currency: str = "USD"

@dataclass
class OrderItem:
    product_id: str
    quantity: int
    unit_price: Money

@dataclass
class Order:
    id: str
    customer_id: str
    items: list[OrderItem]
    total: Money
    status: str = "pending"
    created_at: datetime = field(default_factory=lambda: datetime.now(timezone.utc))

# --- Interfaces (Ports) ---

class OrderRepository(abc.ABC):
    @abc.abstractmethod
    def save(self, order: Order) -> None: ...

    @abc.abstractmethod
    def find_by_id(self, order_id: str) -> Optional[Order]: ...

class PaymentGateway(abc.ABC):
    @abc.abstractmethod
    def charge(self, customer_id: str, amount: Money) -> bool: ...

class NotificationService(abc.ABC):
    @abc.abstractmethod
    def send(self, customer_id: str, message: str) -> None: ...

# --- Implementations (Adapters) ---

class InMemoryOrderRepository(OrderRepository):
    def __init__(self):
        self._orders: dict[str, Order] = {}

    def save(self, order: Order) -> None:
        self._orders[order.id] = order

    def find_by_id(self, order_id: str) -> Optional[Order]:
        return self._orders.get(order_id)

class StripePaymentGateway(PaymentGateway):
    def charge(self, customer_id: str, amount: Money) -> bool:
        print(f"[Stripe] Charging {customer_id}: {amount.amount} {amount.currency}")
        return True  # Simulate success

class EmailNotificationService(NotificationService):
    def send(self, customer_id: str, message: str) -> None:
        print(f"[Email -> {customer_id}] {message}")

# --- Application Service ---

class OrderService:
    def __init__(
        self,
        repo: OrderRepository,
        payments: PaymentGateway,
        notifications: NotificationService,
    ):
        self.repo = repo
        self.payments = payments
        self.notifications = notifications

    def place_order(self, customer_id: str, items: list[OrderItem]) -> Order:
        total_amount = sum(item.unit_price.amount * item.quantity for item in items)
        total = Money(amount=total_amount)

        order = Order(
            id=str(uuid4()),
            customer_id=customer_id,
            items=items,
            total=total,
        )

        # Charge payment
        if not self.payments.charge(customer_id, total):
            order.status = "payment_failed"
            self.repo.save(order)
            raise RuntimeError("Payment failed")

        order.status = "confirmed"
        self.repo.save(order)

        self.notifications.send(
            customer_id,
            f"Order {order.id} confirmed! Total: ${total.amount:.2f}",
        )

        return order

# --- Container Setup ---

def create_production_container() -> punq.Container:
    container = punq.Container()

    # Infrastructure (singletons)
    container.register(OrderRepository, InMemoryOrderRepository,
                       scope=punq.Scope.singleton)
    container.register(PaymentGateway, StripePaymentGateway,
                       scope=punq.Scope.singleton)
    container.register(NotificationService, EmailNotificationService,
                       scope=punq.Scope.singleton)

    # Application services
    container.register(OrderService)

    return container

# --- Usage ---

if __name__ == "__main__":
    container = create_production_container()
    order_service = container.resolve(OrderService)

    items = [
        OrderItem(product_id="SKU-001", quantity=2, unit_price=Money(29.99)),
        OrderItem(product_id="SKU-042", quantity=1, unit_price=Money(49.99)),
    ]

    order = order_service.place_order("customer-123", items)
    print(f"Order {order.id}: {order.status}, total=${order.total.amount:.2f}")
```

### Example 5: Testing with Mock Dependencies

```python
"""
Testing the OrderService from Example 4 with punq and mock dependencies.
"""

import pytest
import punq
from unittest.mock import MagicMock, call

# Import from Example 4
# from myapp.domain import Order, OrderItem, Money
# from myapp.interfaces import OrderRepository, PaymentGateway, NotificationService
# from myapp.services import OrderService

@pytest.fixture
def mock_repo():
    repo = MagicMock(spec=OrderRepository)
    repo.find_by_id.return_value = None
    return repo

@pytest.fixture
def mock_payments():
    gateway = MagicMock(spec=PaymentGateway)
    gateway.charge.return_value = True  # Default: payment succeeds
    return gateway

@pytest.fixture
def mock_notifications():
    return MagicMock(spec=NotificationService)

@pytest.fixture
def test_container(mock_repo, mock_payments, mock_notifications):
    """Build a test container with all mocks injected."""
    container = punq.Container()
    container.register(OrderRepository, instance=mock_repo)
    container.register(PaymentGateway, instance=mock_payments)
    container.register(NotificationService, instance=mock_notifications)
    container.register(OrderService)
    return container

@pytest.fixture
def order_service(test_container):
    return test_container.resolve(OrderService)

class TestOrderService:

    def test_place_order_charges_payment(self, order_service, mock_payments):
        items = [OrderItem("SKU-1", quantity=1, unit_price=Money(10.00))]

        order_service.place_order("cust-1", items)

        mock_payments.charge.assert_called_once_with("cust-1", Money(10.00))

    def test_place_order_saves_to_repo(self, order_service, mock_repo):
        items = [OrderItem("SKU-1", quantity=2, unit_price=Money(5.00))]

        order = order_service.place_order("cust-1", items)

        mock_repo.save.assert_called_once()
        saved_order = mock_repo.save.call_args[0][0]
        assert saved_order.status == "confirmed"
        assert saved_order.total.amount == 10.00

    def test_place_order_sends_notification(
        self, order_service, mock_notifications
    ):
        items = [OrderItem("SKU-1", quantity=1, unit_price=Money(25.00))]

        order = order_service.place_order("cust-1", items)

        mock_notifications.send.assert_called_once()
        args = mock_notifications.send.call_args[0]
        assert args[0] == "cust-1"
        assert "25.00" in args[1]

    def test_place_order_payment_failure(
        self, order_service, mock_payments, mock_repo, mock_notifications
    ):
        mock_payments.charge.return_value = False

        items = [OrderItem("SKU-1", quantity=1, unit_price=Money(100.00))]

        with pytest.raises(RuntimeError, match="Payment failed"):
            order_service.place_order("cust-1", items)

        # Order saved with failed status
        saved_order = mock_repo.save.call_args[0][0]
        assert saved_order.status == "payment_failed"

        # No notification sent on failure
        mock_notifications.send.assert_not_called()

    def test_multiple_items_totaled_correctly(self, order_service, mock_payments):
        items = [
            OrderItem("SKU-1", quantity=2, unit_price=Money(10.00)),
            OrderItem("SKU-2", quantity=3, unit_price=Money(5.00)),
        ]

        order_service.place_order("cust-1", items)

        charged_amount = mock_payments.charge.call_args[0][1]
        assert charged_amount.amount == 35.00  # (2*10) + (3*5)
```

---

## Comparison with Alternatives

| Feature | punq | dependency-injector | injector | Python's built-in |
|---------|------|--------------------|---------|--------------------|
| Lines of code | ~300 | ~15,000+ | ~2,000+ | N/A |
| Dependencies | 0 | Cython (optional) | 0 | N/A |
| Auto-wiring | Yes (type hints) | Yes (wiring module) | Yes (type hints) | Manual |
| Singleton scope | Yes | Yes | Yes | Manual |
| Request scope | No | Yes | Yes | Manual |
| Thread safety | No | Yes | Yes | N/A |
| Async support | No | Yes | No | N/A |
| Learning curve | Minimal | Steep | Moderate | N/A |

**Choose punq when** simplicity and minimalism are your top priorities.
**Choose dependency-injector when** you need production-grade features like thread safety, async, and scoped providers.
**Choose injector when** you want a middle ground with more features than punq but less complexity than dependency-injector.

---

## References

- **GitHub:** https://github.com/bobthemighty/punq
- **PyPI:** https://pypi.org/project/punq/
- **Documentation:** https://punq.readthedocs.io/
- **Source code:** The entire library is in a single `punq.py` file (~300 lines)
