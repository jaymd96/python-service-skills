# punq -- Minimal Dependency Injection Container for Python

> A tiny (~300 lines), opinionated dependency injection container with no dependencies of its own.

## Overview

**punq** is a minimalist dependency injection (DI) container for Python. It provides a
simple, type-driven approach to wiring up application components without the complexity
of larger DI frameworks. The entire library fits in a single file of roughly 300 lines,
making it easy to understand, audit, and debug.

**Key Characteristics:**

- **Version:** 0.7.0 (latest stable release on PyPI)
- **Python:** Requires 3.7+
- **License:** MIT
- **Dependencies:** None (pure Python, zero external dependencies)
- **Size:** ~300 lines of code in a single module
- **Thread Safety:** Not thread-safe by default (see Gotchas section)

**Core Features:**

- Register services by type, factory function, or concrete instance
- Resolve dependencies automatically via type annotations (auto-wiring)
- Singleton and transient lifecycle management
- Factory function support with automatic argument injection
- Nested dependency resolution
- Clear error messages for missing or circular registrations

**When to Use punq:**

- You want dependency injection but not a massive framework
- Your project is small to medium in size
- You prefer explicit registration over classpath scanning or decorators-everywhere patterns
- You value simplicity and readability over feature richness
- You want to easily swap implementations for testing

**When NOT to Use punq:**

- You need advanced features like scoped lifetimes, child containers, or interception/AOP
- You need thread-safe container mutations at runtime
- You need async-aware resolution
- Your project already uses a framework with its own DI (e.g., Django with its settings, or FastAPI's `Depends`)

---

## Installation

```bash
pip install punq
```

Or in your `pyproject.toml`:

```toml
[project]
dependencies = [
    "punq>=0.7.0",
]
```

Or in `requirements.txt`:

```
punq>=0.7.0
```

---

## Core API

### Creating a Container

```python
import punq

container = punq.Container()
```

The `Container` is the central object. You register services into it, then resolve
them out of it. A typical application creates one container at startup and uses it
throughout the application lifecycle.

### Registering Services

The `register` method is the primary way to tell the container how to create objects.
It has several modes of operation.

#### Signature

```python
container.register(
    service,           # The abstract type or interface (usually an ABC or Protocol)
    implementation=None,  # The concrete class, factory, or instance
    instance=None,     # A pre-built instance to always return (singleton)
    factory=None,      # A callable that creates the service
    scope=punq.Scope.dependent,  # Lifecycle scope
)
```

#### Registration by Type (Auto-Wiring)

When you register a concrete class, punq inspects its `__init__` type annotations
to automatically resolve and inject dependencies.

```python
import punq

class UserRepository:
    pass

class UserService:
    def __init__(self, repo: UserRepository):
        self.repo = repo

container = punq.Container()
container.register(UserRepository)
container.register(UserService)

service = container.resolve(UserService)
# punq auto-creates a UserRepository and injects it into UserService
assert isinstance(service.repo, UserRepository)
```

#### Registration with an Interface/Base Class

You can register a concrete type against an abstract base class or protocol.
When resolving, you ask for the abstract type and get the concrete implementation.

```python
import abc
import punq

class MessageSender(abc.ABC):
    @abc.abstractmethod
    def send(self, message: str) -> None:
        pass

class EmailSender(MessageSender):
    def send(self, message: str) -> None:
        print(f"Sending email: {message}")

container = punq.Container()
container.register(MessageSender, EmailSender)

sender = container.resolve(MessageSender)
assert isinstance(sender, EmailSender)
sender.send("Hello!")  # "Sending email: Hello!"
```

#### Registration with a Factory Function

A factory function gives you full control over how an object is created.
punq will auto-inject the factory's annotated parameters.

```python
import punq

class DatabaseConfig:
    def __init__(self):
        self.connection_string = "postgresql://localhost/mydb"

class Database:
    def __init__(self, connection_string: str):
        self.connection_string = connection_string

def create_database(config: DatabaseConfig) -> Database:
    return Database(connection_string=config.connection_string)

container = punq.Container()
container.register(DatabaseConfig)
container.register(Database, factory=create_database)

db = container.resolve(Database)
assert db.connection_string == "postgresql://localhost/mydb"
```

#### Registration with a Specific Instance

If you already have an object you want the container to return every time,
use the `instance` parameter.

```python
import punq

class AppConfig:
    def __init__(self, debug: bool, version: str):
        self.debug = debug
        self.version = version

config = AppConfig(debug=True, version="1.0.0")

container = punq.Container()
container.register(AppConfig, instance=config)

resolved = container.resolve(AppConfig)
assert resolved is config  # Same object
assert resolved.debug is True
```

#### Passing Explicit Constructor Arguments

You can pass keyword arguments to `register` that will be forwarded to the
constructor or factory when the service is resolved.

```python
import punq

class GreetingService:
    def __init__(self, greeting: str, loud: bool):
        self.greeting = greeting
        self.loud = loud

    def greet(self, name: str) -> str:
        msg = f"{self.greeting}, {name}!"
        return msg.upper() if self.loud else msg

container = punq.Container()
container.register(GreetingService, greeting="Hello", loud=True)

svc = container.resolve(GreetingService)
assert svc.greet("World") == "HELLO, WORLD!"
```

### Resolving Services

The `resolve` method looks up a registered service type and returns an instance.

```python
service = container.resolve(ServiceType)
```

If the service depends on other registered types (via type annotations in `__init__`),
those dependencies are resolved recursively.

```python
import punq

class Logger:
    def log(self, msg: str) -> None:
        print(msg)

class Repository:
    def __init__(self, logger: Logger):
        self.logger = logger

class Service:
    def __init__(self, repo: Repository, logger: Logger):
        self.repo = repo
        self.logger = logger

container = punq.Container()
container.register(Logger)
container.register(Repository)
container.register(Service)

# Resolves the full dependency tree: Logger -> Repository -> Service
service = container.resolve(Service)
assert isinstance(service.repo, Repository)
assert isinstance(service.logger, Logger)
assert isinstance(service.repo.logger, Logger)
```

### Singleton vs. Transient Scope

punq supports two lifecycle scopes via the `punq.Scope` enum:

| Scope | Behavior |
|-------|----------|
| `Scope.dependent` | **Default.** A new instance is created every time `resolve` is called. (Transient) |
| `Scope.singleton` | One instance is created on first resolve and reused for all subsequent resolves. |

```python
import punq

class TransientService:
    pass

class SingletonService:
    pass

container = punq.Container()
container.register(TransientService, scope=punq.Scope.dependent)
container.register(SingletonService, scope=punq.Scope.singleton)

# Transient: different instances each time
t1 = container.resolve(TransientService)
t2 = container.resolve(TransientService)
assert t1 is not t2

# Singleton: same instance every time
s1 = container.resolve(SingletonService)
s2 = container.resolve(SingletonService)
assert s1 is s2
```

**Note:** Registering with `instance=` always behaves as a singleton regardless of
the `scope` parameter, since there is only one pre-built instance to return.

---

## API Quick Reference

```
# punq API Reference
# Version: 0.7.0 | Python: 3.7+

## Container

# Create a container
container = punq.Container()

## Registering Services

# Register a concrete type (auto-wiring)
container.register(ConcreteClass)

# Register interface -> implementation
container.register(AbstractBase, ConcreteImpl)

# Register with a factory function
container.register(ServiceType, factory=my_factory_func)

# Register a pre-built instance (always singleton)
container.register(ServiceType, instance=my_instance)

# Register as singleton
container.register(ServiceType, ConcreteImpl, scope=punq.Scope.singleton)

# Register as transient (default)
container.register(ServiceType, ConcreteImpl, scope=punq.Scope.dependent)

# Register with explicit constructor kwargs
container.register(ServiceType, kwarg1="value1", kwarg2=42)

## Resolving Services

# Resolve a service (creates or retrieves instance)
instance = container.resolve(ServiceType)

## Scopes

punq.Scope.dependent   # New instance per resolve (transient, default)
punq.Scope.singleton   # One instance, reused forever

## Exceptions

punq.MissingDependencyError    # Raised when a dependency cannot be resolved
punq.InvalidRegistrationError  # Raised when registration is invalid
```

---

## Scoping

punq supports two scopes natively:

1. **`Scope.dependent` (transient)** -- the default. Every call to `resolve()` creates
   a fresh instance. Dependencies are also freshly resolved.

2. **`Scope.singleton`** -- the first call to `resolve()` creates and caches the instance.
   All subsequent calls return the cached instance. Singleton dependencies are shared
   across all consumers.

punq does **not** support request-scoped, session-scoped, or thread-scoped lifetimes
natively. If you need request scoping (common in web applications), you can implement it
by creating a new child container per request or by managing a dictionary of instances
yourself.

### Mixing Scopes

Be careful when a singleton depends on a transient. The singleton is created once,
so it captures one instance of the transient dependency forever:

```python
import punq

class RequestContext:
    """Meant to be transient, one per request."""
    pass

class CacheService:
    """Meant to be singleton, shared across requests."""
    def __init__(self, ctx: RequestContext):
        self.ctx = ctx

container = punq.Container()
container.register(RequestContext, scope=punq.Scope.dependent)
container.register(CacheService, scope=punq.Scope.singleton)

svc1 = container.resolve(CacheService)
svc2 = container.resolve(CacheService)

# Same CacheService, same RequestContext -- the transient is "captured"
assert svc1 is svc2
assert svc1.ctx is svc2.ctx  # Same RequestContext instance!
```

This is a well-known DI anti-pattern called the **Captive Dependency** problem.
Avoid letting singletons depend on transients.

---

## Integration Patterns

### With FastAPI

FastAPI has its own `Depends` system, but punq can manage your service layer while
FastAPI handles HTTP-level dependencies.

```python
import punq
from fastapi import FastAPI, Depends

# --- Domain Layer ---

class UserRepository:
    def get_user(self, user_id: int) -> dict:
        return {"id": user_id, "name": "Alice"}

class UserService:
    def __init__(self, repo: UserRepository):
        self.repo = repo

    def get_user(self, user_id: int) -> dict:
        return self.repo.get_user(user_id)

# --- Container Setup ---

def create_container() -> punq.Container:
    container = punq.Container()
    container.register(UserRepository, scope=punq.Scope.singleton)
    container.register(UserService, scope=punq.Scope.singleton)
    return container

container = create_container()

# --- FastAPI Integration ---

app = FastAPI()

def get_user_service() -> UserService:
    return container.resolve(UserService)

@app.get("/users/{user_id}")
def read_user(user_id: int, service: UserService = Depends(get_user_service)):
    return service.get_user(user_id)
```

### With Flask

```python
import punq
from flask import Flask, g, jsonify

# --- Domain Layer ---

class OrderRepository:
    def get_order(self, order_id: int) -> dict:
        return {"id": order_id, "total": 99.99}

class OrderService:
    def __init__(self, repo: OrderRepository):
        self.repo = repo

    def get_order(self, order_id: int) -> dict:
        return self.repo.get_order(order_id)

# --- Container Setup ---

def create_container() -> punq.Container:
    container = punq.Container()
    container.register(OrderRepository)
    container.register(OrderService)
    return container

# --- Flask Integration ---

app = Flask(__name__)
container = create_container()

@app.route("/orders/<int:order_id>")
def get_order(order_id: int):
    service = container.resolve(OrderService)
    return jsonify(service.get_order(order_id))
```

### With Django (Manual Wiring)

```python
# myproject/container.py
import punq
from myapp.repositories import DjangoUserRepository
from myapp.services import UserService
from myapp.interfaces import UserRepositoryInterface

def create_container() -> punq.Container:
    container = punq.Container()
    container.register(UserRepositoryInterface, DjangoUserRepository,
                       scope=punq.Scope.singleton)
    container.register(UserService, scope=punq.Scope.singleton)
    return container

container = create_container()

# myapp/views.py
from myproject.container import container
from myapp.services import UserService

def user_detail_view(request, user_id):
    service = container.resolve(UserService)
    user = service.get_user(user_id)
    # ... render response
```

---

## Testing -- Dependency Replacement

One of the primary motivations for using DI is testability. With punq, you can
create a test container that substitutes real implementations with fakes or mocks.

### Strategy 1: Separate Test Container

```python
import punq
from unittest.mock import MagicMock

# Production interfaces and implementations
class EmailService:
    def send(self, to: str, body: str) -> None:
        # Real email sending logic
        pass

class NotificationService:
    def __init__(self, email: EmailService):
        self.email = email

    def notify_user(self, user_email: str, message: str) -> None:
        self.email.send(user_email, message)

# Production container
def create_container() -> punq.Container:
    container = punq.Container()
    container.register(EmailService, scope=punq.Scope.singleton)
    container.register(NotificationService)
    return container

# Test container with mocks
def create_test_container() -> punq.Container:
    container = punq.Container()
    mock_email = MagicMock(spec=EmailService)
    container.register(EmailService, instance=mock_email)
    container.register(NotificationService)
    return container

# In tests
def test_notification_sends_email():
    container = create_test_container()
    svc = container.resolve(NotificationService)

    svc.notify_user("alice@example.com", "Hello!")

    mock_email = container.resolve(EmailService)
    mock_email.send.assert_called_once_with("alice@example.com", "Hello!")
```

### Strategy 2: Override Specific Registrations

Since punq allows you to re-register a service (the last registration wins),
you can start from a production container and selectively override:

```python
import punq
from unittest.mock import MagicMock

class PaymentGateway:
    def charge(self, amount: float) -> bool:
        # Calls real payment API
        return True

class OrderProcessor:
    def __init__(self, gateway: PaymentGateway):
        self.gateway = gateway

    def process(self, amount: float) -> str:
        if self.gateway.charge(amount):
            return "success"
        return "failed"

def create_container() -> punq.Container:
    container = punq.Container()
    container.register(PaymentGateway, scope=punq.Scope.singleton)
    container.register(OrderProcessor)
    return container

def test_order_processing():
    container = create_container()

    # Override the payment gateway with a mock
    mock_gateway = MagicMock(spec=PaymentGateway)
    mock_gateway.charge.return_value = True
    container.register(PaymentGateway, instance=mock_gateway)

    processor = container.resolve(OrderProcessor)
    result = processor.process(49.99)

    assert result == "success"
    mock_gateway.charge.assert_called_once_with(49.99)
```

### Strategy 3: Pytest Fixtures

```python
import pytest
import punq
from unittest.mock import MagicMock

class DatabaseConnection:
    def query(self, sql: str) -> list:
        # Real database query
        return []

class UserRepository:
    def __init__(self, db: DatabaseConnection):
        self.db = db

    def find_user(self, user_id: int) -> dict:
        rows = self.db.query(f"SELECT * FROM users WHERE id = {user_id}")
        return rows[0] if rows else {}

class UserService:
    def __init__(self, repo: UserRepository):
        self.repo = repo

    def get_user(self, user_id: int) -> dict:
        return self.repo.find_user(user_id)

@pytest.fixture
def container():
    """Create a test container with mocked database."""
    c = punq.Container()

    mock_db = MagicMock(spec=DatabaseConnection)
    mock_db.query.return_value = [{"id": 1, "name": "Alice"}]

    c.register(DatabaseConnection, instance=mock_db)
    c.register(UserRepository)
    c.register(UserService)
    return c

def test_get_user(container):
    service = container.resolve(UserService)
    user = service.get_user(1)
    assert user["name"] == "Alice"

def test_user_not_found(container):
    # Override the mock for this specific test
    mock_db = container.resolve(DatabaseConnection)
    mock_db.query.return_value = []

    service = container.resolve(UserService)
    user = service.get_user(999)
    assert user == {}
```

---

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
