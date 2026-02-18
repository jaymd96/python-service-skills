# transitions — API Reference

> Part of the transitions skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Core Concepts](#core-concepts)
- [Core API](#core-api)
  - [Machine -- Creating a State Machine](#machine----creating-a-state-machine)
  - [States](#states)
  - [Transitions](#transitions)
  - [Triggering Transitions](#triggering-transitions)
  - [Checking State](#checking-state)
  - [Auto-Transitions](#auto-transitions)
  - [The EventData Object](#the-eventdata-object)
- [Quick Reference](#quick-reference)

## Core Concepts

The library is built around five key concepts:

1. **States** -- Distinct modes or phases an object can be in.
2. **Transitions** -- Rules that define how the object moves between states, triggered by events.
3. **Model** -- The object whose state is being managed. Can be any Python object.
4. **Machine** -- The orchestrator that manages states, transitions, and callbacks.
5. **Triggers** -- Named events that initiate transitions. Each trigger becomes a method on the model.

The simplest possible state machine:

```python
from transitions import Machine

class Matter:
    pass

model = Matter()

machine = Machine(
    model=model,
    states=["solid", "liquid", "gas"],
    initial="solid",
)

# The machine auto-generates trigger methods, state checks, and auto-transitions
model.state          # 'solid'
model.is_solid()     # True
model.to_liquid()    # auto-transition: moves to 'liquid'
model.state          # 'liquid'
```

---

## Core API

### `Machine` -- Creating a State Machine

```python
from transitions import Machine

Machine(
    model=None,                   # Object(s) to manage; None = Machine itself is the model
    states=[],                    # List of state names, State objects, Enums, or dicts
    transitions=[],               # List of transition dicts, lists, or Transition objects
    initial='initial',            # Starting state name (or None to defer)
    auto_transitions=True,        # Generate to_<state>() methods automatically
    send_event=False,             # Wrap trigger args in an EventData object
    queued=False,                 # Queue events for sequential processing
    ignore_invalid_triggers=False,# Suppress MachineError for invalid triggers
    before_state_change=None,     # Global callback(s) before every state change
    after_state_change=None,      # Global callback(s) after every state change
    prepare_event=None,           # Callback(s) before transition evaluation
    finalize_event=None,          # Callback(s) after transition (always runs, even on error)
    on_exception=None,            # Callback(s) when an exception is raised during transition
    on_final=None,                # Callback(s) when entering a state marked final=True
    model_attribute='state',      # Name of the state attribute on the model
    model_override=False,         # If True, only override methods already defined on model
)
```

**Key points:**

- If `model=None`, the `Machine` instance itself becomes the model. This is convenient for simple cases.
- Multiple models can share a single machine by passing a list: `model=[obj1, obj2]`.
- The `model_attribute` parameter lets you use a name other than `.state` (e.g., `model_attribute='status'`).

### States

States can be defined in three ways:

```python
from transitions import Machine, State

# 1. As plain strings (simplest)
states = ['pending', 'running', 'failed']

# 2. As State objects (full control)
states = [
    State(name='pending', on_enter=['log_pending']),
    State(name='running', on_enter=['log_running'], on_exit=['cleanup_running']),
    State(name='failed', on_enter=['alert_failure'], final=True),
]

# 3. As dictionaries (equivalent to State objects)
states = [
    {'name': 'pending', 'on_enter': 'log_pending'},
    {'name': 'running', 'on_enter': 'log_running'},
    {'name': 'failed', 'on_enter': 'alert_failure', 'final': True},
]
```

#### `State` Object Parameters

```python
State(
    name='running',                    # Required: unique state identifier
    on_enter=['callback1', 'callback2'],  # Callback(s) when entering this state
    on_exit=['callback'],              # Callback(s) when exiting this state
    ignore_invalid_triggers=False,     # State-level override for invalid trigger behavior
    final=False,                       # Mark as a terminal/final state
    tags=['active', 'billable'],       # Custom metadata tags (accessible via State.tags)
)
```

#### Using Enums as States

```python
from enum import Enum
from transitions import Machine

class TrafficState(Enum):
    RED = 1
    YELLOW = 2
    GREEN = 3

machine = Machine(
    states=TrafficState,
    initial=TrafficState.RED,
)
```

When using enums, `model.state` returns the enum member (e.g., `TrafficState.RED`) rather than a string.

#### Adding States Dynamically

```python
machine.add_states(['maintenance', 'decommissioned'])
machine.add_states([State(name='archived', final=True)])
```

### Transitions

Transitions define the rules for state changes. Each transition has a trigger (event name), a source state, and a destination state.

```python
# Dictionary form (recommended for readability)
transitions = [
    {
        'trigger': 'heat',           # Event name -- becomes a method on the model
        'source': 'solid',           # Origin state; '*' = any state
        'dest': 'liquid',            # Destination state; None = internal (no state change);
                                     #   '=' = reflexive (re-enter same state)
        'conditions': 'is_hot',      # Condition callback(s); must return True to proceed
        'unless': 'is_frozen',       # Inverse condition(s); must return False to proceed
        'before': 'start_melting',   # Callback(s) executed before the state change
        'after': 'notify_melted',    # Callback(s) executed after the state change
        'prepare': 'check_temp',     # Callback(s) before condition evaluation
    },
]

# List/tuple form (compact)
transitions = [
    # [trigger, source, dest]
    ['heat', 'solid', 'liquid'],
    ['cool', 'liquid', 'solid'],
    ['heat', 'liquid', 'gas'],
]
```

#### Special Source and Destination Values

| Value | Meaning |
|-------|---------|
| `'*'` (source) | Matches any current state (wildcard) |
| `None` (dest) | Internal transition -- callbacks fire but the state does not change |
| `'='` (dest) | Reflexive transition -- exits and re-enters the current state |

```python
transitions = [
    # From any state, 'reset' goes to 'idle'
    {'trigger': 'reset', 'source': '*', 'dest': 'idle'},

    # Internal transition: fires callbacks but state stays the same
    {'trigger': 'check', 'source': 'running', 'dest': None, 'after': 'run_health_check'},

    # Reflexive: exits and re-enters the state (on_exit and on_enter fire)
    {'trigger': 'refresh', 'source': 'running', 'dest': '=', 'after': 'reload_config'},
]
```

#### Multiple Source States

```python
# A single trigger can have multiple source states
{'trigger': 'fail', 'source': ['running', 'installing', 'degraded'], 'dest': 'failed'}
```

### Triggering Transitions

Once a machine is configured, trigger methods are auto-generated on the model:

```python
from transitions import Machine

class Order:
    pass

order = Order()
machine = Machine(
    model=order,
    states=['pending', 'paid', 'shipped', 'delivered'],
    transitions=[
        {'trigger': 'pay', 'source': 'pending', 'dest': 'paid'},
        {'trigger': 'ship', 'source': 'paid', 'dest': 'shipped'},
        {'trigger': 'deliver', 'source': 'shipped', 'dest': 'delivered'},
    ],
    initial='pending',
)

# Method 1: Call the auto-generated trigger method directly
order.pay()
order.state  # 'paid'

# Method 2: Use the trigger() method with the event name as a string
order.trigger('ship')
order.state  # 'shipped'
```

Trigger methods return `True` if the transition succeeded, or `False` if conditions prevented it (when `ignore_invalid_triggers=True`). If the trigger is invalid for the current state and `ignore_invalid_triggers=False` (the default), a `MachineError` is raised.

### Checking State

```python
# Current state (string or enum value)
order.state                       # 'paid'

# Auto-generated boolean check methods
order.is_paid()                   # True
order.is_pending()                # False

# Check if a trigger is currently allowed
order.may_ship()                  # True (we're in 'paid', and 'ship' goes from 'paid')
order.may_pay()                   # False (we're no longer in 'pending')

# Get available triggers from a specific state
machine.get_triggers('paid')      # ['ship', 'to_pending', 'to_paid', ...]

# Get the State object for the current state
machine.get_state(order.state)    # <State('paid')@...>
```

### Auto-Transitions

By default (`auto_transitions=True`), the machine generates `to_<state>()` methods for every state. These let you jump directly to any state:

```python
order.to_pending()   # Jump to 'pending' from any state
order.to_delivered() # Jump to 'delivered' from any state
```

To disable auto-transitions:

```python
machine = Machine(..., auto_transitions=False)
```

### The `EventData` Object

When `send_event=True`, all callbacks receive an `EventData` object instead of raw `*args, **kwargs`:

```python
machine = Machine(model=obj, ..., send_event=True)

def my_callback(event):
    event.state       # Source State object
    event.transition  # Transition object
    event.machine     # Machine instance
    event.model       # Model instance
    event.args        # Positional arguments from the trigger call
    event.kwargs      # Keyword arguments from the trigger call
    event.trigger     # Trigger name (string)
    event.error       # Exception object (in on_exception/finalize callbacks)
```

---

## Quick Reference

### State Definition

| Form | Example |
|------|---------|
| String | `'running'` |
| State object | `State(name='running', on_enter=['cb'], final=True)` |
| Dictionary | `{'name': 'running', 'on_enter': 'cb'}` |
| Enum | `class S(Enum): RUNNING = 1` |

### Transition Definition

| Key | Type | Description |
|-----|------|-------------|
| `trigger` | `str` | Event name (becomes a method on the model) |
| `source` | `str` or `list[str]` | Source state(s); `'*'` for any |
| `dest` | `str` or `None` | Destination state; `None` = internal; `'='` = reflexive |
| `conditions` | `str` or `list[str]` | Must all return `True` |
| `unless` | `str` or `list[str]` | Must all return `False` |
| `before` | `str` or `list[str]` | Callback(s) before state change |
| `after` | `str` or `list[str]` | Callback(s) after state change |
| `prepare` | `str` or `list[str]` | Callback(s) before condition evaluation |

### Model Methods (Auto-Generated)

| Method | Description |
|--------|-------------|
| `model.state` | Current state (str or Enum) |
| `model.trigger('event')` | Trigger a transition by name |
| `model.<trigger_name>()` | Trigger a transition directly |
| `model.is_<state>()` | Check if model is in a specific state |
| `model.may_<trigger>()` | Check if a trigger is allowed from current state |
| `model.to_<state>()` | Auto-transition to a state (if `auto_transitions=True`) |
