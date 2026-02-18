# Blinker — Examples & Gotchas

> Part of the blinker skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Weak Reference Garbage Collection](#1-weak-reference-garbage-collection-lambdas-and-local-functions)
  - [2. Forgetting **kwargs in Receiver Signature](#2-forgetting-kwargs-in-receiver-signature)
  - [3. Sender Identity Uses is, Not ==](#3-sender-identity-uses-is-not-)
  - [4. Signal Creation in Modules vs Instances](#4-signal-creation-in-modules-vs-instances)
  - [5. Thread Safety](#5-thread-safety)
  - [6. Memory Leaks with Strong References](#6-memory-leaks-with-strong-references)
  - [7. Exception Handling in Receivers](#7-exception-handling-in-receivers)
  - [8. send() with Async Receivers](#8-send-with-async-receivers)
- [Complete Code Examples](#complete-code-examples)
  - [Example 1: Basic Signal / Connect / Send](#example-1-basic-signal--connect--send)
  - [Example 2: Sender Filtering](#example-2-sender-filtering)
  - [Example 3: Decorator Syntax](#example-3-decorator-syntax)
  - [Example 4: Real-World Audit Logging](#example-4-real-world-audit-logging)
  - [Example 5: Integration with transitions for State Machine Events](#example-5-integration-with-transitions-for-state-machine-events)
  - [Example 6: Testing with Signals](#example-6-testing-with-signals)
- [References](#references)

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

## References

- **Documentation:** https://blinker.readthedocs.io/
- **GitHub:** https://github.com/pallets-eco/blinker
- **PyPI:** https://pypi.org/project/blinker/
- **Changelog:** https://github.com/pallets-eco/blinker/blob/main/CHANGES.rst
- **Flask Signals Docs:** https://flask.palletsprojects.com/en/stable/signals/
