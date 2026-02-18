---
name: transitions
description: Lightweight finite state machine (FSM) library. Use when implementing state machines, defining states and transitions with callbacks, using hierarchical or async state machines, or adding workflow state management. Triggers on state machine, FSM, transitions, states, workflow states, HierarchicalMachine, AsyncMachine.
---

# transitions — Finite State Machines (v0.9.2)

## Quick Start

```bash
pip install transitions
```

```python
from transitions import Machine

class Order:
    pass

model = Order()
machine = Machine(
    model=model,
    states=["pending", "paid", "shipped", "delivered"],
    transitions=[
        {"trigger": "pay", "source": "pending", "dest": "paid"},
        {"trigger": "ship", "source": "paid", "dest": "shipped"},
        {"trigger": "deliver", "source": "shipped", "dest": "delivered"},
    ],
    initial="pending",
)

model.pay()      # pending → paid
model.state      # "paid"
model.may_ship() # True
```

## Key Patterns

### Callbacks and conditions
```python
transitions = [
    {
        "trigger": "pay",
        "source": "pending",
        "dest": "paid",
        "before": "validate_payment",
        "after": "send_receipt",
        "conditions": ["has_valid_card"],
    }
]
# Callback order: prepare → conditions → before → exit_state → enter_state → after
```

### State callbacks
```python
states = [
    {"name": "paid", "on_enter": "log_payment", "on_exit": "archive_payment"},
]
```

### Wildcard and internal transitions
```python
{"trigger": "error", "source": "*", "dest": "failed"}       # from any state
{"trigger": "log", "source": "active", "dest": None}         # internal (no state change)
{"trigger": "refresh", "source": "active", "dest": "="}      # reflexive (exit+enter)
```

## References

- **[api.md](references/api.md)** — Machine class, basic transitions, triggers, auto_transitions
- **[states.md](references/states.md)** — State objects, callbacks (on_enter/on_exit), conditions, ordered transitions
- **[advanced.md](references/advanced.md)** — HierarchicalMachine, GraphMachine, AsyncMachine, threading, extensions
- **[examples.md](references/examples.md)** — Complete examples, gotchas, state machine patterns

## Grep Patterns

- `Machine\(` — Find state machine definitions
- `states=|transitions=` — Find machine configuration
- `on_enter_|on_exit_` — Find state callbacks
- `HierarchicalMachine|GraphMachine` — Find advanced machines
