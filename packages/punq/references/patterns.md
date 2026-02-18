# punq — Patterns

> Part of the punq skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Scoping](#scoping)
  - [Mixing Scopes](#mixing-scopes)
- [Integration Patterns](#integration-patterns)
  - [With FastAPI](#with-fastapi)
  - [With Flask](#with-flask)
  - [With Django (Manual Wiring)](#with-django-manual-wiring)
- [Testing -- Dependency Replacement](#testing----dependency-replacement)
  - [Strategy 1: Separate Test Container](#strategy-1-separate-test-container)
  - [Strategy 2: Override Specific Registrations](#strategy-2-override-specific-registrations)
  - [Strategy 3: Pytest Fixtures](#strategy-3-pytest-fixtures)

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
