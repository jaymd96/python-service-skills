---
name: blinker
description: Fast signal/event dispatching for Python (observer pattern). Use when implementing event-driven architecture, decoupling components with signals, adding pub/sub within a process, or broadcasting state changes. Triggers on signals, events, observer pattern, blinker, pub/sub, event-driven, signal.send, signal.connect.
---

# blinker — Signal/Event Dispatching (v1.9.0)

## Quick Start

```bash
pip install blinker
```

```python
from blinker import signal

order_placed = signal("order-placed")

@order_placed.connect
def handle_order(sender, **kwargs):
    print(f"Order from {sender}: {kwargs}")

order_placed.send("checkout", order_id=123, total=49.99)
```

## Key Patterns

### Connect and send
```python
from blinker import Signal

order_placed = Signal()

@order_placed.connect
def handle(sender, **kwargs):
    print(f"Order {kwargs['order_id']} from {sender}")

results = order_placed.send("checkout", order_id=123)
# returns [(receiver_func, return_value), ...]
```

### Sender filtering
```python
@order_placed.connect_via(specific_sender)
def premium_handler(sender, **kwargs):
    pass  # only called when sender is specific_sender
```

### Temporary connections (testing)
```python
with order_placed.connected_to(mock_handler):
    order_placed.send("test")  # mock_handler receives it
# disconnected automatically
```

## References

- **[api.md](references/api.md)** — Signal class, Namespace, connect/disconnect/send API
- **[patterns.md](references/patterns.md)** — Sender filtering, decorator syntax, temporary connections, weak references
- **[advanced.md](references/advanced.md)** — Async support, threading, Flask integration, custom subclasses
- **[examples.md](references/examples.md)** — Complete examples, gotchas, common mistakes

## Grep Patterns

- `Signal\(` — Find signal definitions
- `\.connect\(|\.send\(` — Find signal usage
- `@.*\.connect` — Find decorator-style connections
