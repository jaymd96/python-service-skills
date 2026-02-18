# blinker

> Fast, simple in-process signal/event dispatching for Python (observer pattern).

`blinker` provides a fast dispatcher mechanism for decoupled communication between components in a Python application. It implements the observer pattern: one part of your code emits a signal, and any number of receivers (handlers) respond to it -- without the emitter needing to know who is listening. Maintained by the Pallets Community Ecosystem (the same organization behind Flask, Jinja2, and Click).

| Detail | Value |
|---|---|
| **PyPI** | [blinker](https://pypi.org/project/blinker/) |
| **Latest stable version** | **1.9.0** (November 2024) |
| **Python** | 3.9+ |
| **Source** | [github.com/pallets-eco/blinker](https://github.com/pallets-eco/blinker) |
| **Docs** | [blinker.readthedocs.io](https://blinker.readthedocs.io) |
| **License** | MIT |
| **Dependencies** | None (pure Python) |
| **Thread Safety** | Yes |

---

## Table of Contents

1. [Installation](#installation)
2. [Core Concepts](#core-concepts)
3. [Core API](#core-api)
   - [Creating Signals](#creating-signals)
   - [Connecting Receivers](#connecting-receivers)
   - [Sending Signals](#sending-signals)
   - [Disconnecting Receivers](#disconnecting-receivers)
   - [Introspection](#introspection)
   - [Context Managers](#context-managers)
   - [Namespaces](#namespaces)
   - [Meta-Signals](#meta-signals)
4. [Sender Filtering](#sender-filtering)
5. [Weak References](#weak-references)
6. [Anonymous Signals vs Named Signals](#anonymous-signals-vs-named-signals)
7. [Async Support](#async-support)
8. [Integration Patterns](#integration-patterns)
9. [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
10. [Complete Code Examples](#complete-code-examples)

---

## Installation

```bash
pip install blinker
```

Or in `pyproject.toml`:

```toml
[project]
dependencies = [
    "blinker>=1.9.0",
]
```

Blinker has **zero runtime dependencies**. It is pure Python.

---

## Core Concepts

Blinker implements the **observer pattern** (also called publish-subscribe or event-driven pattern) for in-process communication:

- **Signal** -- an event channel that can be emitted (sent) and listened to (connected)
- **Sender** -- the object that emits the signal; receivers can filter by sender
- **Receiver** -- a callable (function, method, or any callable) that responds when a signal is sent
- **Payload** -- arbitrary keyword arguments passed alongside the signal

The flow is always: **define signal** -> **connect receivers** -> **send signal** -> **receivers execute**.

---

## Core API

### Creating Signals

There are two ways to create signals: anonymous (class-based) and named (global registry).

#### Anonymous signals with `Signal()`

```python
from blinker import Signal

# Create a standalone signal
on_user_login = Signal()

# With documentation string
on_user_login = Signal(doc="Emitted when a user successfully logs in")
```

Anonymous signals are unique instances. Two calls to `Signal()` produce two distinct, unrelated signals. Use these when you want signals scoped to a module, class, or component.

```python
class UserService:
    on_created = Signal(doc="Emitted when a new user is created")
    on_deleted = Signal(doc="Emitted when a user is deleted")
    on_updated = Signal(doc="Emitted when a user is modified")
```

#### Named signals with `signal()`

```python
from blinker import signal

# Get or create a named signal from the global registry
my_signal = signal("user-login")

# Calling signal() with the same name returns the same object
same_signal = signal("user-login")
assert my_signal is same_signal  # True
```

Named signals enable decoupled modules to communicate without importing a shared signal object. Module A emits `signal("order-completed")` and Module B connects to `signal("order-completed")` -- they never import each other.

```python
my_signal = signal("my-signal", doc="Description for documentation purposes")
```

---

### Connecting Receivers

A receiver is any callable with the signature `def receiver(sender, **kwargs)`.

#### Basic connection

```python
from blinker import Signal

order_placed = Signal()

def handle_order(sender, **kwargs):
    print(f"Order placed by {sender}: {kwargs}")

order_placed.connect(handle_order)
```

#### Decorator syntax with `@signal.connect`

```python
@order_placed.connect
def log_order(sender, **kwargs):
    print(f"Logging order from {sender}")
```

The decorator returns the function unchanged, so the function is still callable normally.

#### Decorator with sender filter: `@signal.connect_via(sender)`

```python
class PremiumService:
    pass

premium = PremiumService()

@order_placed.connect_via(premium)
def handle_premium_order(sender, **kwargs):
    """Only called when sender is the premium service instance."""
    print(f"Premium order: {kwargs}")
```

#### Connection parameters

```python
signal.connect(
    receiver,           # callable with signature (sender, **kwargs)
    sender=ANY,         # only receive from this sender (default: any sender)
    weak=True,          # use weak reference (default: True)
)
```

---

### Sending Signals

#### `signal.send(sender, **kwargs)`

Send a signal synchronously. All connected receivers are called in the order they were connected.

```python
order_placed = Signal()

@order_placed.connect
def handler_a(sender, **kwargs):
    return "result_a"

@order_placed.connect
def handler_b(sender, **kwargs):
    return "result_b"

# Send the signal
results = order_placed.send("my_app", order_id="ORD-123", total=99.99)

# results is a list of (receiver, return_value) tuples
for receiver, value in results:
    print(f"{receiver.__name__} returned {value}")
# handler_a returned result_a
# handler_b returned result_b
```

The `sender` argument identifies who is emitting the signal. It is typically `self` in a method, a class, or a module-level sentinel.

```python
class OrderEngine:
    def place_order(self, order):
        # self is the sender
        order_placed.send(self, order_id=order.id, total=order.total)
```

#### `signal.send_async(sender, **kwargs)`

Send a signal asynchronously. Async receivers are awaited; sync receivers are called normally.

```python
results = await order_placed.send_async(self, order_id="ORD-456")
```

See the [Async Support](#async-support) section for details.

---

### Disconnecting Receivers

#### `signal.disconnect(receiver, sender=ANY)`

Remove a receiver from the signal.

```python
def my_handler(sender, **kwargs):
    print("handled")

order_placed.connect(my_handler)

# Later, disconnect it
order_placed.disconnect(my_handler)
```

To disconnect a receiver that was connected to a specific sender:

```python
order_placed.connect(my_handler, sender=specific_sender)
order_placed.disconnect(my_handler, sender=specific_sender)
```

---

### Introspection

#### `signal.receivers`

A mapping of receiver identities. Check if any receivers are connected:

```python
if order_placed.receivers:
    # At least one receiver is connected
    expensive_data = compute_payload()
    order_placed.send(self, data=expensive_data)
```

This is an important optimization: avoid computing expensive payloads when nobody is listening.

#### `signal.has_receivers_for(sender)`

Check whether any receivers would be notified for a given sender:

```python
if order_placed.has_receivers_for(premium_service):
    order_placed.send(premium_service, order_id="ORD-789")
```

This accounts for both global receivers (connected with `sender=ANY`) and sender-specific receivers.

#### `signal.receivers_for(sender)`

Iterate over all receiver callables that would be notified for a given sender:

```python
for receiver in order_placed.receivers_for(my_sender):
    print(f"Would notify: {receiver}")
```

---

### Context Managers

#### `signal.connected_to(receiver, sender=ANY)`

Temporarily connect a receiver for the duration of a block. The receiver is automatically disconnected when the block exits.

```python
def temporary_handler(sender, **kwargs):
    print(f"Temporary: {kwargs}")

with order_placed.connected_to(temporary_handler):
    order_placed.send("app", order_id="ORD-001")  # temporary_handler is called

order_placed.send("app", order_id="ORD-002")  # temporary_handler is NOT called
```

This is especially useful in tests:

```python
def test_order_emits_signal():
    captured = []

    with order_placed.connected_to(lambda s, **kw: captured.append(kw)):
        service.place_order(order_data)

    assert len(captured) == 1
    assert captured[0]["order_id"] == "ORD-123"
```

#### `signal.muted()`

Temporarily suppress all receivers. No receivers are called while the signal is muted.

```python
with order_placed.muted():
    # None of the connected receivers will fire
    order_placed.send("app", order_id="ORD-003")
```

Useful in tests when you want to call code that emits signals without triggering side effects:

```python
def test_order_creation_logic():
    with order_placed.muted(), order_failed.muted():
        result = service.place_order(data)
    assert result.status == "created"
```

---

### Namespaces

#### `Namespace()`

A `Namespace` is an isolated registry of named signals. Calling `namespace.signal(name)` returns the same signal object for the same name within that namespace.

```python
from blinker import Namespace

# Create an isolated namespace
my_ns = Namespace()

# Get or create signals within the namespace
sig_a = my_ns.signal("event-a")
sig_b = my_ns.signal("event-b")

# Same name returns the same object
assert my_ns.signal("event-a") is sig_a  # True
```

The top-level `signal()` function uses a default global namespace (`blinker.base.default_namespace`). Custom namespaces let you avoid collisions:

```python
from blinker import Namespace

# Plugin A's signals
plugin_a_ns = Namespace()
plugin_a_ready = plugin_a_ns.signal("ready")

# Plugin B's signals -- no collision even with same name
plugin_b_ns = Namespace()
plugin_b_ready = plugin_b_ns.signal("ready")

assert plugin_a_ready is not plugin_b_ready  # different signals
```

---

### Meta-Signals

Every `Signal` instance has two meta-signals that fire when receivers connect or disconnect:

```python
from blinker import Signal

my_signal = Signal()

@my_signal.receiver_connected
def on_receiver_connected(receiver, **kwargs):
    print(f"New receiver connected: {receiver}")

@my_signal.receiver_disconnected
def on_receiver_disconnected(receiver, **kwargs):
    print(f"Receiver disconnected: {receiver}")
```

These are useful for debugging, monitoring, or implementing lazy initialization patterns (e.g., only start a background task when the first receiver connects).

---

## Sender Filtering

By default, a connected receiver hears signals from **any** sender. You can restrict a receiver to only respond to signals from a specific sender.

```python
from blinker import Signal

data_saved = Signal()

class UserRepository:
    def save(self, user):
        data_saved.send(self, entity=user, entity_type="user")

class OrderRepository:
    def save(self, order):
        data_saved.send(self, entity=order, entity_type="order")

user_repo = UserRepository()
order_repo = OrderRepository()

# This handler only fires when user_repo is the sender
@data_saved.connect_via(user_repo)
def on_user_saved(sender, **kwargs):
    print(f"User saved: {kwargs['entity']}")

# This handler fires for ALL senders
@data_saved.connect
def on_any_save(sender, **kwargs):
    print(f"Something saved via {sender}: {kwargs['entity_type']}")

user_repo.save({"name": "Alice"})
# Output:
#   User saved: {'name': 'Alice'}
#   Something saved via <UserRepository>: user

order_repo.save({"id": "ORD-1"})
# Output:
#   Something saved via <OrderRepository>: order
# (on_user_saved is NOT called)
```

Sender filtering uses **identity** (`is`), not equality (`==`). The sender you connect with must be the exact same object as the sender passed to `send()`. See the [gotchas section](#3-sender-identity-uses-is-not-) for implications.

The `ANY` sentinel (importable from `blinker`) matches all senders and is the default:

```python
from blinker import ANY

data_saved.connect(handler, sender=ANY)  # same as data_saved.connect(handler)
```

---

## Weak References

By default, blinker holds **weak references** to receivers. This means:

- When a receiver function is garbage collected (e.g., goes out of scope), it is automatically disconnected from the signal.
- You do not need to manually disconnect receivers that are short-lived.
- Module-level functions and class methods referenced elsewhere are not affected because other references keep them alive.

### The weak reference default

```python
from blinker import Signal

my_signal = Signal()

def setup():
    def local_handler(sender, **kwargs):
        print("handled")

    my_signal.connect(local_handler)
    # local_handler is connected via a weak reference

setup()
# local_handler has gone out of scope and been garbage collected
# It is automatically disconnected from my_signal

my_signal.send("test")  # local_handler is NOT called
```

### Strong references with `weak=False`

Use `weak=False` when you need a receiver to stay connected even when no other references to it exist:

```python
my_signal.connect(lambda sender, **kwargs: print("fired"), weak=False)
# The lambda has no other references, but it stays connected
# because blinker holds a strong reference

my_signal.send("test")  # "fired" is printed
```

### When to use `weak=False`

- Lambdas (they have no other reference)
- Closures created in a function that will go out of scope
- Dynamically created handler functions
- Any receiver where you cannot guarantee another reference will keep it alive

### When weak references (default) are fine

- Module-level functions (the module keeps the reference)
- Bound methods on long-lived objects
- Functions assigned to variables that persist

---

## Anonymous Signals vs Named Signals

Blinker offers two patterns for creating signals. Choose the right one based on your architecture.

### Anonymous signals (`Signal()`)

```python
from blinker import Signal

# Each call creates a unique, independent signal
on_login = Signal(doc="User logged in")
on_logout = Signal(doc="User logged out")
```

**Use when:**
- Signals are defined alongside the component that emits them
- You have a central module that exports signal instances
- Both emitter and receiver can import the signal object directly
- You want compile-time/import-time discovery of signals

**Pattern:**
```python
# signals.py
from blinker import Signal

user_created = Signal()
user_deleted = Signal()

# emitter.py
from signals import user_created

def create_user(data):
    user = save_to_db(data)
    user_created.send(current_app, user=user)

# handler.py
from signals import user_created

@user_created.connect
def send_welcome_email(sender, **kwargs):
    email.send(kwargs["user"].email, "Welcome!")
```

### Named signals (`signal()`)

```python
from blinker import signal

# Returns the same object for the same name
login_signal = signal("user-login")
```

**Use when:**
- Emitter and receiver are in separate packages that should not import each other
- You are building a plugin system where plugins register for signals by name
- You want string-based signal discovery (e.g., from configuration)
- You need true decoupling between packages

**Pattern:**
```python
# package_a/emitter.py (no import from package_b)
from blinker import signal

def create_user(data):
    user = save_to_db(data)
    signal("user-created").send(current_app, user=user)

# package_b/handler.py (no import from package_a)
from blinker import signal

@signal("user-created").connect
def send_welcome_email(sender, **kwargs):
    email.send(kwargs["user"].email, "Welcome!")
```

### Recommendation

Prefer **anonymous signals** for most cases. They provide better IDE support, static analysis, and import-time error detection. Use **named signals** when you genuinely need cross-package decoupling or plugin architectures.

---

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

---

## Gotchas and Common Mistakes

### 1. Weak Reference Garbage Collection (Lambdas and Local Functions)

This is the most common pitfall. Since blinker uses weak references by default, receivers with no other references are silently garbage collected.

```python
from blinker import Signal

my_signal = Signal()

# BUG: lambda has no other reference, gets garbage collected immediately
my_signal.connect(lambda sender, **kw: print("fired"))
my_signal.send("test")  # NOTHING HAPPENS -- lambda was already collected

# BUG: local function defined in a function that returns
def setup_handlers():
    def handler(sender, **kwargs):
        print("handled")
    my_signal.connect(handler)
    # handler goes out of scope when setup_handlers() returns

setup_handlers()
my_signal.send("test")  # NOTHING HAPPENS -- handler was garbage collected
```

**Fix:** Use `weak=False` for receivers with no persistent reference:

```python
# FIX: strong reference keeps the lambda alive
my_signal.connect(lambda sender, **kw: print("fired"), weak=False)

# FIX: strong reference for local function
def setup_handlers():
    def handler(sender, **kwargs):
        print("handled")
    my_signal.connect(handler, weak=False)
```

**Alternatively**, keep a reference to the receiver:

```python
# FIX: module-level variable holds the reference
_handler = lambda sender, **kw: print("fired")
my_signal.connect(_handler)  # weak=True is fine, _handler persists
```

**Safe by default:** Module-level functions and decorated functions are safe because the module holds a reference:

```python
# SAFE: module-level function is referenced by the module
@my_signal.connect
def my_handler(sender, **kwargs):
    print("This works fine")
```

### 2. Forgetting `**kwargs` in Receiver Signature

Receivers **must** accept `**kwargs` to be forward-compatible. If a signal adds new keyword arguments later, receivers without `**kwargs` will raise `TypeError`.

```python
# BAD: breaks if signal payload changes
@my_signal.connect
def fragile_handler(sender, order_id):
    print(order_id)

# GOOD: forward-compatible
@my_signal.connect
def robust_handler(sender, **kwargs):
    print(kwargs.get("order_id"))

# ALSO GOOD: explicit args you need, plus **kwargs for the rest
@my_signal.connect
def mixed_handler(sender, order_id=None, **kwargs):
    print(order_id)
```

### 3. Sender Identity Uses `is`, Not `==`

Sender filtering uses Python's `is` operator (identity), not `==` (equality). This means:

```python
from blinker import Signal

my_signal = Signal()

# These are different objects even though they're "equal"
sender_a = "hello"
sender_b = "".join(["h", "e", "l", "l", "o"])
# sender_a == sender_b is True, but sender_a is sender_b may be False

# Connecting to sender_a
@my_signal.connect_via(sender_a)
def handler(sender, **kwargs):
    print("received")

my_signal.send(sender_b)  # MAY NOT trigger handler if sender_b is not sender_a
```

**Practical implications:**
- Use object instances (e.g., `self`, class instances) as senders, not strings or numbers
- Strings may or may not be interned by Python, making identity checks unreliable
- If you must use strings, use a single defined constant:

```python
# SAFE: use a constant that both sides reference
APP_SENDER = "my-app"

my_signal.connect(handler, sender=APP_SENDER)
my_signal.send(APP_SENDER, data="value")  # same object, identity check passes
```

### 4. Signal Creation in Modules vs Instances

Define signals at the **module or class level**, not inside `__init__` or methods. Creating signals inside `__init__` means each instance has its own signal, which defeats the purpose of shared signaling.

```python
# BAD: each instance gets a different signal
class BadService:
    def __init__(self):
        self.on_complete = Signal()  # New signal per instance

svc1 = BadService()
svc2 = BadService()
# svc1.on_complete is NOT svc2.on_complete -- connecting to one misses the other

# GOOD: signal is shared across all instances
class GoodService:
    on_complete = Signal()  # Class-level, shared by all instances

svc1 = GoodService()
svc2 = GoodService()
# svc1.on_complete is svc2.on_complete -- same signal
# Use sender filtering to distinguish instances if needed
```

### 5. Thread Safety

Blinker's `send()` is thread-safe for dispatch: multiple threads can call `send()` on the same signal concurrently. However:

- **Connecting and disconnecting** while another thread is sending can cause issues. Register receivers during application startup before any signals are sent.
- **Receiver execution** is in the calling thread. If you need thread-safe handlers, the handler itself must handle synchronization.
- **Receivers run sequentially** in the thread that calls `send()`. A slow receiver blocks subsequent receivers and the sender.

```python
# SAFE pattern: register at startup, send from any thread
import threading
from blinker import Signal

event = Signal()

# Register during startup (single-threaded)
@event.connect
def handler(sender, **kwargs):
    # This runs in whatever thread calls event.send()
    # Use locks if accessing shared mutable state
    with some_lock:
        shared_list.append(kwargs["data"])

# Multiple threads can send safely
def worker(thread_id):
    event.send(thread_id, data=f"from thread {thread_id}")

threads = [threading.Thread(target=worker, args=(i,)) for i in range(10)]
for t in threads:
    t.start()
```

### 6. Memory Leaks with Strong References

Using `weak=False` keeps receivers alive indefinitely. If you create and connect receivers dynamically (e.g., in a loop or per-request), you will leak memory:

```python
# MEMORY LEAK: strong-referenced handlers accumulate
for i in range(10000):
    my_signal.connect(
        lambda sender, n=i, **kw: process(n),
        weak=False,  # never garbage collected, never disconnected
    )
```

**Fix:** Explicitly disconnect strong-referenced receivers when done, or use `connected_to()` for temporary connections:

```python
# SAFE: temporary connection
with my_signal.connected_to(my_handler):
    do_work()
# my_handler is automatically disconnected
```

### 7. Exception Handling in Receivers

If a receiver raises an exception, it propagates to the sender and **prevents subsequent receivers from running**.

```python
@my_signal.connect
def bad_handler(sender, **kwargs):
    raise RuntimeError("oops")

@my_signal.connect
def good_handler(sender, **kwargs):
    print("This never runs if bad_handler raises first")

my_signal.send("app")  # RuntimeError propagates to caller
```

**Fix:** Catch exceptions in receivers:

```python
@my_signal.connect
def safe_handler(sender, **kwargs):
    try:
        risky_operation(kwargs)
    except Exception as e:
        logger.exception(f"Handler error: {e}")
        # Don't re-raise -- let other handlers run
```

### 8. `send()` with Async Receivers

If you call `send()` (synchronous) and an async receiver is connected, the async receiver will **not** be awaited. It returns a coroutine object instead of executing.

```python
async def async_handler(sender, **kwargs):
    await do_something()

my_signal.connect(async_handler)

# BUG: async_handler returns a coroutine, does NOT execute
results = my_signal.send("app")
# results[0][1] is a coroutine object, not the handler's return value

# FIX: use send_async in async contexts
results = await my_signal.send_async("app")
```

---

## Complete Code Examples

### Example 1: Basic Signal / Connect / Send

```python
from blinker import Signal

# 1. Define a signal
order_completed = Signal(doc="Emitted when an order is fulfilled")

# 2. Define receivers
def send_confirmation_email(sender, **kwargs):
    print(f"Sending email for order {kwargs['order_id']} to {kwargs['email']}")
    return "email_sent"

def update_inventory(sender, **kwargs):
    for item in kwargs.get("items", []):
        print(f"Reducing stock for {item['sku']} by {item['qty']}")
    return "inventory_updated"

def record_metrics(sender, **kwargs):
    print(f"Recording: order_total={kwargs['total']}")
    return "metrics_recorded"

# 3. Connect receivers
order_completed.connect(send_confirmation_email)
order_completed.connect(update_inventory)
order_completed.connect(record_metrics)

# 4. Send the signal
results = order_completed.send(
    "order_service",
    order_id="ORD-12345",
    email="alice@example.com",
    total=149.99,
    items=[
        {"sku": "WIDGET-A", "qty": 2},
        {"sku": "GADGET-B", "qty": 1},
    ],
)

# 5. Inspect results
print("\nResults:")
for receiver, return_value in results:
    print(f"  {receiver.__name__}: {return_value}")

# Output:
# Sending email for order ORD-12345 to alice@example.com
# Reducing stock for WIDGET-A by 2
# Reducing stock for GADGET-B by 1
# Recording: order_total=149.99
#
# Results:
#   send_confirmation_email: email_sent
#   update_inventory: inventory_updated
#   record_metrics: metrics_recorded
```

### Example 2: Sender Filtering

```python
from blinker import Signal

data_saved = Signal()

class PostgresRepo:
    name = "postgres"
    def save(self, record):
        print(f"  [Postgres] Saving {record}")
        data_saved.send(self, record=record, store="postgres")

class RedisCache:
    name = "redis"
    def save(self, record):
        print(f"  [Redis] Caching {record}")
        data_saved.send(self, record=record, store="redis")

pg = PostgresRepo()
redis = RedisCache()

# Handler for ALL saves
@data_saved.connect
def log_all_saves(sender, **kwargs):
    print(f"  [Audit] Save to {kwargs['store']}: {kwargs['record']}")

# Handler ONLY for postgres saves
@data_saved.connect_via(pg)
def on_postgres_save(sender, **kwargs):
    print(f"  [PG Hook] Post-save trigger for: {kwargs['record']}")

print("--- Postgres save ---")
pg.save({"id": 1, "name": "Alice"})

print("\n--- Redis save ---")
redis.save({"id": 1, "name": "Alice"})

# Output:
# --- Postgres save ---
#   [Postgres] Saving {'id': 1, 'name': 'Alice'}
#   [Audit] Save to postgres: {'id': 1, 'name': 'Alice'}
#   [PG Hook] Post-save trigger for: {'id': 1, 'name': 'Alice'}
#
# --- Redis save ---
#   [Redis] Caching {'id': 1, 'name': 'Alice'}
#   [Audit] Save to redis: {'id': 1, 'name': 'Alice'}
```

### Example 3: Decorator Syntax

```python
from blinker import Signal

# Signal as a class attribute
class TaskQueue:
    task_submitted = Signal()
    task_completed = Signal()
    task_failed = Signal()

    def submit(self, task_name, payload):
        self.task_submitted.send(self, task_name=task_name, payload=payload)
        try:
            result = self._execute(task_name, payload)
            self.task_completed.send(self, task_name=task_name, result=result)
            return result
        except Exception as e:
            self.task_failed.send(
                self, task_name=task_name, error=str(e), error_type=type(e).__name__
            )
            raise

    def _execute(self, task_name, payload):
        # Simulate work
        if task_name == "fail":
            raise ValueError("Intentional failure")
        return {"status": "done", "task": task_name}

# Use decorator syntax to register handlers
@TaskQueue.task_submitted.connect
def on_task_submitted(sender, **kwargs):
    print(f"Task submitted: {kwargs['task_name']}")

@TaskQueue.task_completed.connect
def on_task_completed(sender, **kwargs):
    print(f"Task completed: {kwargs['task_name']} -> {kwargs['result']}")

@TaskQueue.task_failed.connect
def on_task_failed(sender, **kwargs):
    print(f"Task FAILED: {kwargs['task_name']} ({kwargs['error_type']}: {kwargs['error']})")

# Use it
queue = TaskQueue()
queue.submit("process_report", {"report_id": 42})

try:
    queue.submit("fail", {})
except ValueError:
    pass

# Output:
# Task submitted: process_report
# Task completed: process_report -> {'status': 'done', 'task': 'process_report'}
# Task submitted: fail
# Task FAILED: fail (ValueError: Intentional failure)
```

### Example 4: Real-World Audit Logging

```python
"""
Complete audit logging system using blinker signals.
Captures all mutations with structured context for compliance.
"""

import json
import logging
from datetime import datetime, timezone
from dataclasses import dataclass, asdict
from typing import Any
from blinker import Signal

# --- Signal Definitions ---

class AuditSignals:
    """Central registry of auditable events."""
    record_created = Signal(doc="A database record was created")
    record_updated = Signal(doc="A database record was updated")
    record_deleted = Signal(doc="A database record was deleted")
    auth_success = Signal(doc="Authentication succeeded")
    auth_failure = Signal(doc="Authentication failed")
    permission_denied = Signal(doc="Authorization check failed")

# --- Audit Log Handler ---

@dataclass
class AuditEntry:
    timestamp: str
    event: str
    actor: str
    entity_type: str | None = None
    entity_id: str | None = None
    details: dict[str, Any] | None = None
    ip_address: str | None = None

audit_logger = logging.getLogger("audit")

def _create_audit_entry(event_name: str, kwargs: dict) -> AuditEntry:
    return AuditEntry(
        timestamp=datetime.now(timezone.utc).isoformat(),
        event=event_name,
        actor=kwargs.get("actor", "system"),
        entity_type=kwargs.get("entity_type"),
        entity_id=str(kwargs.get("entity_id", "")),
        details=kwargs.get("details"),
        ip_address=kwargs.get("ip_address"),
    )

@AuditSignals.record_created.connect
def audit_record_created(sender, **kwargs):
    entry = _create_audit_entry("record_created", kwargs)
    audit_logger.info(json.dumps(asdict(entry)))

@AuditSignals.record_updated.connect
def audit_record_updated(sender, **kwargs):
    entry = _create_audit_entry("record_updated", kwargs)
    audit_logger.info(json.dumps(asdict(entry)))

@AuditSignals.record_deleted.connect
def audit_record_deleted(sender, **kwargs):
    entry = _create_audit_entry("record_deleted", kwargs)
    audit_logger.warning(json.dumps(asdict(entry)))

@AuditSignals.auth_failure.connect
def audit_auth_failure(sender, **kwargs):
    entry = _create_audit_entry("auth_failure", kwargs)
    audit_logger.error(json.dumps(asdict(entry)))

@AuditSignals.permission_denied.connect
def audit_permission_denied(sender, **kwargs):
    entry = _create_audit_entry("permission_denied", kwargs)
    audit_logger.error(json.dumps(asdict(entry)))

# --- Usage in Application Code ---

class UserRepository:
    def create(self, data: dict, actor: str):
        user_id = save_to_database(data)  # hypothetical
        AuditSignals.record_created.send(
            self,
            actor=actor,
            entity_type="user",
            entity_id=user_id,
            details={"username": data["username"], "role": data.get("role")},
        )
        return user_id

    def update(self, user_id: str, changes: dict, actor: str):
        old_record = load_from_database(user_id)  # hypothetical
        apply_changes(user_id, changes)            # hypothetical
        AuditSignals.record_updated.send(
            self,
            actor=actor,
            entity_type="user",
            entity_id=user_id,
            details={"changes": changes, "previous": old_record},
        )

    def delete(self, user_id: str, actor: str):
        delete_from_database(user_id)  # hypothetical
        AuditSignals.record_deleted.send(
            self,
            actor=actor,
            entity_type="user",
            entity_id=user_id,
        )
```

### Example 5: Integration with `transitions` for State Machine Events

This example combines blinker signals with the `transitions` state machine library to emit signals on state transitions:

```python
"""
State machine with blinker signal integration.
Emits signals on every state transition for cross-cutting concerns.
"""

from blinker import Signal
from transitions import Machine

# --- Signal Definitions ---

state_entered = Signal(doc="Emitted when a state machine enters a new state")
state_exited = Signal(doc="Emitted when a state machine exits a state")
transition_completed = Signal(doc="Emitted after a state transition completes")
transition_denied = Signal(doc="Emitted when a transition is denied by a condition")

# --- State Machine Model ---

class Order:
    """Order with state machine and blinker signal integration."""

    states = ["pending", "confirmed", "processing", "shipped", "delivered", "cancelled"]

    transitions = [
        {"trigger": "confirm", "source": "pending", "dest": "confirmed"},
        {"trigger": "process", "source": "confirmed", "dest": "processing"},
        {"trigger": "ship", "source": "processing", "dest": "shipped"},
        {"trigger": "deliver", "source": "shipped", "dest": "delivered"},
        {"trigger": "cancel", "source": ["pending", "confirmed"], "dest": "cancelled"},
    ]

    def __init__(self, order_id: str, customer: str):
        self.order_id = order_id
        self.customer = customer

        self.machine = Machine(
            model=self,
            states=Order.states,
            transitions=Order.transitions,
            initial="pending",
            after_state_change="on_state_change",
            send_event=True,
        )

    def on_state_change(self, event):
        """Called by transitions after every state change.
        Bridges transitions callbacks into blinker signals."""
        transition_completed.send(
            self,
            order_id=self.order_id,
            customer=self.customer,
            trigger=event.event.name,
            source=event.transition.source,
            dest=event.transition.dest,
            state=self.state,
        )

# --- Signal Handlers ---

@transition_completed.connect
def log_transition(sender, **kwargs):
    print(
        f"[{kwargs['order_id']}] "
        f"{kwargs['source']} -> {kwargs['dest']} "
        f"(trigger: {kwargs['trigger']})"
    )

@transition_completed.connect
def notify_on_shipment(sender, **kwargs):
    if kwargs["dest"] == "shipped":
        print(f"  Notifying {kwargs['customer']}: your order has shipped!")

@transition_completed.connect
def alert_on_cancellation(sender, **kwargs):
    if kwargs["dest"] == "cancelled":
        print(f"  ALERT: Order {kwargs['order_id']} was cancelled from {kwargs['source']}")

# --- Usage ---

order = Order("ORD-100", "alice@example.com")
order.confirm()
order.process()
order.ship()
order.deliver()

print()

order2 = Order("ORD-101", "bob@example.com")
order2.cancel()

# Output:
# [ORD-100] pending -> confirmed (trigger: confirm)
# [ORD-100] confirmed -> processing (trigger: process)
# [ORD-100] processing -> shipped (trigger: ship)
#   Notifying alice@example.com: your order has shipped!
# [ORD-100] shipped -> delivered (trigger: deliver)
#
# [ORD-101] pending -> cancelled (trigger: cancel)
#   ALERT: Order ORD-101 was cancelled from pending
```

### Example 6: Testing with Signals

```python
"""
Patterns for testing code that uses blinker signals.
"""

import pytest
from blinker import Signal

# The signal and code under test
order_placed = Signal()

class OrderService:
    def place_order(self, order_data):
        order_id = f"ORD-{order_data['item']}"
        order_placed.send(self, order_id=order_id, total=order_data["price"])
        return order_id

# --- Test Patterns ---

class TestOrderService:
    def test_signal_is_emitted_with_correct_data(self):
        """Use connected_to() to capture and verify signal payloads."""
        captured = []

        def capture(sender, **kwargs):
            captured.append({"sender": sender, **kwargs})

        service = OrderService()

        with order_placed.connected_to(capture):
            result = service.place_order({"item": "widget", "price": 9.99})

        assert result == "ORD-widget"
        assert len(captured) == 1
        assert captured[0]["order_id"] == "ORD-widget"
        assert captured[0]["total"] == 9.99
        assert isinstance(captured[0]["sender"], OrderService)

    def test_signal_emitted_from_specific_sender(self):
        """Use sender filtering to test specific sender behavior."""
        service_a = OrderService()
        service_b = OrderService()
        captured_from_a = []

        def capture(sender, **kwargs):
            captured_from_a.append(kwargs)

        order_placed.connect(capture, sender=service_a)

        try:
            service_a.place_order({"item": "from_a", "price": 10})
            service_b.place_order({"item": "from_b", "price": 20})

            assert len(captured_from_a) == 1
            assert captured_from_a[0]["order_id"] == "ORD-from_a"
        finally:
            order_placed.disconnect(capture, sender=service_a)

    def test_muted_signal_prevents_side_effects(self):
        """Use muted() when testing logic without signal side effects."""
        side_effects = []

        @order_placed.connect
        def side_effect_handler(sender, **kwargs):
            side_effects.append("side_effect_triggered")

        service = OrderService()

        with order_placed.muted():
            result = service.place_order({"item": "test", "price": 5})

        assert result == "ORD-test"
        assert len(side_effects) == 0  # no side effects fired

        order_placed.disconnect(side_effect_handler)

    def test_return_values_from_receivers(self):
        """Verify that send() collects return values from all receivers."""
        def validator(sender, **kwargs):
            if kwargs["total"] > 1000:
                return {"valid": False, "reason": "Amount too high"}
            return {"valid": True}

        def enricher(sender, **kwargs):
            return {"tax": kwargs["total"] * 0.1}

        service = OrderService()

        with order_placed.connected_to(validator):
            with order_placed.connected_to(enricher):
                results = order_placed.send(
                    service, order_id="ORD-test", total=500
                )

        values = [v for _, v in results]
        assert {"valid": True} in values
        assert {"tax": 50.0} in values
```

---

## API Quick Reference

```
blinker.Signal(doc=None)
    .connect(receiver, sender=ANY, weak=True) -> receiver
    .connect_via(sender, weak=False) -> decorator
    .disconnect(receiver, sender=ANY) -> None
    .send(sender=None, **kwargs) -> [(receiver, return_value), ...]
    .send_async(sender=None, **kwargs) -> [(receiver, return_value), ...]
    .has_receivers_for(sender) -> bool
    .receivers_for(sender) -> [receiver, ...]
    .receivers -> dict
    .connected_to(receiver, sender=ANY) -> context_manager
    .muted() -> context_manager
    .receiver_connected -> Signal  (meta-signal)
    .receiver_disconnected -> Signal  (meta-signal)

blinker.signal(name, doc=None) -> NamedSignal
    Returns the same NamedSignal for the same name (global registry).

blinker.Namespace()
    .signal(name, doc=None) -> NamedSignal
    Isolated signal registry. Same name -> same signal within namespace.

blinker.ANY
    Sentinel for "any sender" in connect() and has_receivers_for().
```

---

## References

- **Documentation:** https://blinker.readthedocs.io/
- **GitHub:** https://github.com/pallets-eco/blinker
- **PyPI:** https://pypi.org/project/blinker/
- **Changelog:** https://github.com/pallets-eco/blinker/blob/main/CHANGES.rst
- **Flask Signals Docs:** https://flask.palletsprojects.com/en/stable/signals/
