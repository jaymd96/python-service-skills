# Blinker — Advanced Features

> Part of the blinker skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Async Support](#async-support)
- [Integration Patterns](#integration-patterns)
  - [Flask Signals](#flask-signals-built-on-blinker)
  - [Event-Driven Architecture](#event-driven-architecture)
  - [Decoupling Components](#decoupling-components)
  - [Audit Logging Pattern](#audit-logging-pattern)

## Async Support

Blinker 1.6+ introduced `send_async()` for dispatching signals to asynchronous receivers.

### `signal.send_async(sender, **kwargs)`

```python
import asyncio
from blinker import Signal

order_placed = Signal()

async def async_handler(sender, **kwargs):
    await asyncio.sleep(0.1)  # simulate async I/O
    print(f"Async handler: {kwargs['order_id']}")
    return "async_result"

def sync_handler(sender, **kwargs):
    print(f"Sync handler: {kwargs['order_id']}")
    return "sync_result"

order_placed.connect(async_handler)
order_placed.connect(sync_handler)

async def main():
    # send_async() awaits async receivers and calls sync receivers normally
    results = await order_placed.send_async("app", order_id="ORD-001")
    for receiver, value in results:
        print(f"  {receiver.__name__} -> {value}")

asyncio.run(main())
```

**Key behavior:**
- `send_async()` **must** be called with `await` inside an async context
- Async receivers are awaited; sync receivers are called normally
- Returns the same `list[(receiver, return_value)]` structure as `send()`
- If you use `send()` (synchronous) with async receivers connected, the async receivers will **not** be awaited -- they return coroutine objects instead of results

### Mixing sync and async handlers

A common pattern is to have both sync and async receivers on the same signal:

```python
from blinker import Signal

data_changed = Signal()

# Sync receiver for logging
@data_changed.connect
def log_change(sender, **kwargs):
    logger.info(f"Data changed: {kwargs}")

# Async receiver for external API call
async def notify_external(sender, **kwargs):
    async with httpx.AsyncClient() as client:
        await client.post("https://hooks.example.com", json=kwargs)

data_changed.connect(notify_external)

# In async context, use send_async to properly handle both
async def update_data(data):
    save(data)
    await data_changed.send_async(current_app, data=data)
```

---

## Integration Patterns

### Flask Signals (Built on Blinker)

Flask's signal system is built directly on blinker. Flask provides built-in signals and makes it easy to define custom ones.

```python
from flask import Flask
from flask.signals import request_started, request_finished, got_request_exception

app = Flask(__name__)

# Connect to Flask's built-in signals
@request_started.connect_via(app)
def on_request_started(sender, **kwargs):
    print(f"Request started on {sender.name}")

@request_finished.connect_via(app)
def on_request_finished(sender, response, **kwargs):
    print(f"Request finished: {response.status_code}")

@got_request_exception.connect_via(app)
def on_exception(sender, exception, **kwargs):
    print(f"Exception caught: {exception}")

# Define custom signals for your Flask app
from blinker import Signal

user_logged_in = Signal()
user_logged_out = Signal()

@app.route("/login", methods=["POST"])
def login():
    user = authenticate(request.form)
    user_logged_in.send(app, user=user)
    return redirect("/dashboard")
```

Flask's built-in signals include: `request_started`, `request_finished`, `request_tearing_down`, `got_request_exception`, `template_rendered`, `before_render_template`, `appcontext_tearing_down`, `appcontext_pushed`, `appcontext_popped`, `message_flashed`.

### Event-Driven Architecture

Use blinker to build an event-driven architecture where components communicate through signals rather than direct method calls:

```python
from blinker import Signal

# Central signal definitions
class DomainEvents:
    order_placed = Signal(doc="An order was placed")
    payment_received = Signal(doc="Payment was confirmed")
    inventory_reserved = Signal(doc="Inventory was reserved for an order")
    order_shipped = Signal(doc="An order was shipped")
    order_failed = Signal(doc="An order processing step failed")

# Each service connects to the signals it cares about
class PaymentService:
    def __init__(self):
        DomainEvents.order_placed.connect(self.process_payment)

    def process_payment(self, sender, **kwargs):
        order_id = kwargs["order_id"]
        try:
            charge = self.charge_card(kwargs["payment_info"])
            DomainEvents.payment_received.send(
                self, order_id=order_id, charge_id=charge.id
            )
        except PaymentError as e:
            DomainEvents.order_failed.send(
                self, order_id=order_id, reason=str(e), phase="payment"
            )

class InventoryService:
    def __init__(self):
        DomainEvents.payment_received.connect(self.reserve_inventory)

    def reserve_inventory(self, sender, **kwargs):
        order_id = kwargs["order_id"]
        # Reserve stock, then emit next event in the chain
        DomainEvents.inventory_reserved.send(
            self, order_id=order_id, warehouse="WH-01"
        )

class NotificationService:
    def __init__(self):
        DomainEvents.order_shipped.connect(self.notify_customer)
        DomainEvents.order_failed.connect(self.notify_support)

    def notify_customer(self, sender, **kwargs):
        send_email(kwargs["customer_email"], "Your order has shipped!")

    def notify_support(self, sender, **kwargs):
        send_alert(f"Order {kwargs['order_id']} failed: {kwargs['reason']}")
```

### Decoupling Components

Use signals to eliminate hard dependencies between modules:

```python
# Without blinker -- tight coupling
class UserService:
    def __init__(self, email_service, analytics, cache):
        self.email = email_service
        self.analytics = analytics
        self.cache = cache

    def create_user(self, data):
        user = User(**data)
        db.session.add(user)
        db.session.commit()
        self.email.send_welcome(user)        # hard dependency
        self.analytics.track("user_created")  # hard dependency
        self.cache.invalidate("user_list")    # hard dependency
        return user

# With blinker -- decoupled
from blinker import Signal

user_created = Signal()

class UserService:
    def create_user(self, data):
        user = User(**data)
        db.session.add(user)
        db.session.commit()
        user_created.send(self, user=user)  # fire and forget
        return user

# Handlers registered elsewhere (or in separate packages)
@user_created.connect
def send_welcome_email(sender, **kwargs):
    email_service.send_welcome(kwargs["user"])

@user_created.connect
def track_user_created(sender, **kwargs):
    analytics.track("user_created", user_id=kwargs["user"].id)

@user_created.connect
def invalidate_user_cache(sender, **kwargs):
    cache.invalidate("user_list")
```

The `UserService` no longer knows about email, analytics, or caching. New side effects can be added without modifying the service.

### Audit Logging Pattern

A cross-cutting audit trail using signals:

```python
import json
import logging
from datetime import datetime, timezone
from blinker import Signal

# Define auditable signals
entity_created = Signal(doc="Any entity was created")
entity_updated = Signal(doc="Any entity was updated")
entity_deleted = Signal(doc="Any entity was deleted")
access_denied = Signal(doc="An access control check failed")

audit_logger = logging.getLogger("audit")

@entity_created.connect
def audit_creation(sender, **kwargs):
    audit_logger.info(json.dumps({
        "event": "entity_created",
        "entity_type": kwargs.get("entity_type"),
        "entity_id": kwargs.get("entity_id"),
        "actor": kwargs.get("actor"),
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "sender": type(sender).__name__,
    }))

@entity_updated.connect
def audit_update(sender, **kwargs):
    audit_logger.info(json.dumps({
        "event": "entity_updated",
        "entity_type": kwargs.get("entity_type"),
        "entity_id": kwargs.get("entity_id"),
        "changes": kwargs.get("changes"),
        "actor": kwargs.get("actor"),
        "timestamp": datetime.now(timezone.utc).isoformat(),
    }))

@entity_deleted.connect
def audit_deletion(sender, **kwargs):
    audit_logger.warning(json.dumps({
        "event": "entity_deleted",
        "entity_type": kwargs.get("entity_type"),
        "entity_id": kwargs.get("entity_id"),
        "actor": kwargs.get("actor"),
        "timestamp": datetime.now(timezone.utc).isoformat(),
    }))

@access_denied.connect
def audit_access_denied(sender, **kwargs):
    audit_logger.error(json.dumps({
        "event": "access_denied",
        "resource": kwargs.get("resource"),
        "actor": kwargs.get("actor"),
        "required_permission": kwargs.get("required_permission"),
        "timestamp": datetime.now(timezone.utc).isoformat(),
    }))

# Usage from any service
class DocumentService:
    def create_document(self, data, actor):
        doc = Document(**data)
        db.session.add(doc)
        db.session.commit()
        entity_created.send(
            self,
            entity_type="document",
            entity_id=doc.id,
            actor=actor,
        )
        return doc
```
