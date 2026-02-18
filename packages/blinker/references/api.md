# Blinker — API Reference

> Part of the blinker skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Core Concepts](#core-concepts)
- [Core API](#core-api)
  - [Creating Signals](#creating-signals)
  - [Connecting Receivers](#connecting-receivers)
  - [Sending Signals](#sending-signals)
  - [Disconnecting Receivers](#disconnecting-receivers)
  - [Introspection](#introspection)
  - [Context Managers](#context-managers)
  - [Namespaces](#namespaces)
  - [Meta-Signals](#meta-signals)
- [API Quick Reference](#api-quick-reference)

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
