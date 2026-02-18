# transitions — States & Transitions

> Part of the transitions skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [States](#states)
  - [State Object Parameters](#state-object-parameters)
  - [Using Enums as States](#using-enums-as-states)
  - [Adding States Dynamically](#adding-states-dynamically)
- [Callbacks](#callbacks)
  - [Callback Types and Execution Order](#callback-types-and-execution-order)
  - [State Callbacks: on_enter and on_exit](#state-callbacks-on_enter_state-and-on_exit_state)
  - [Transition Callbacks: before, after, prepare](#transition-callbacks-before-after-prepare)
  - [Conditions and Guards: conditions, unless](#conditions-and-guards-conditions-unless)
  - [Global Callbacks on the Machine](#global-callbacks-on-the-machine)
  - [Passing Arguments to Callbacks](#passing-arguments-to-callbacks)
- [Ordered Transitions and Special Values](#ordered-transitions-and-special-values)

## States

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

### `State` Object Parameters

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

### Using Enums as States

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

### Adding States Dynamically

```python
machine.add_states(['maintenance', 'decommissioned'])
machine.add_states([State(name='archived', final=True)])
```

---

## Callbacks

Callbacks are the primary mechanism for executing side effects during state transitions. They can be specified as:

- A **string** naming a method on the model (e.g., `'log_entry'`)
- A **callable** (function, lambda, bound method)
- A **list** of the above (all will be called in order)

### Callback Types and Execution Order

The full callback execution order during a transition is:

```
1.  Machine.prepare_event          -- Before any evaluation
2.  Transition.prepare             -- Before condition checks
3.  Transition.conditions / unless -- May halt the transition
4.  Machine.before_state_change    -- Global pre-change hook
5.  Transition.before              -- Transition-specific pre-change
6.  State.on_exit (source)         -- Leaving the source state
7.  *** STATE CHANGES ***
8.  State.on_enter (destination)   -- Entering the destination state
9.  Transition.after               -- Transition-specific post-change
10. State.on_final (machine)       -- If destination is marked final=True
11. Machine.after_state_change     -- Global post-change hook
12. Machine.finalize_event         -- Always runs (even if an exception occurred)
```

### State Callbacks: `on_enter_<state>` and `on_exit_<state>`

State callbacks fire whenever the state is entered or exited, regardless of which transition caused the change.

```python
from transitions import Machine, State

class Alarm:
    def on_enter_armed(self):
        print("Alarm is now armed")

    def on_exit_armed(self):
        print("Alarm is being disarmed")

    def on_enter_triggered(self):
        print("ALARM TRIGGERED! Notifying authorities.")

alarm = Alarm()
states = [
    State(name='disarmed'),
    State(name='armed', on_enter=['on_enter_armed'], on_exit=['on_exit_armed']),
    State(name='triggered', on_enter=['on_enter_triggered']),
]
transitions = [
    {'trigger': 'arm', 'source': 'disarmed', 'dest': 'armed'},
    {'trigger': 'trigger_alarm', 'source': 'armed', 'dest': 'triggered'},
    {'trigger': 'disarm', 'source': ['armed', 'triggered'], 'dest': 'disarmed'},
]

machine = Machine(model=alarm, states=states, transitions=transitions, initial='disarmed')

alarm.arm()
# Output: Alarm is now armed

alarm.trigger_alarm()
# Output: Alarm is being disarmed
# Output: ALARM TRIGGERED! Notifying authorities.
```

Alternatively, define `on_enter_<state>` / `on_exit_<state>` methods directly on the model and the machine will discover them automatically:

```python
class Alarm:
    def on_enter_armed(self):
        print("Armed")

    def on_exit_armed(self):
        print("Leaving armed")

# No need to specify callbacks in State definitions --
# the Machine finds on_enter_armed and on_exit_armed automatically.
machine = Machine(model=alarm, states=['disarmed', 'armed'], ...)
```

### Transition Callbacks: `before`, `after`, `prepare`

```python
transitions = [
    {
        'trigger': 'deploy',
        'source': 'pending',
        'dest': 'deploying',
        'prepare': 'validate_config',   # Runs first, before conditions
        'conditions': 'resources_available',
        'before': 'reserve_resources',   # Runs after conditions pass, before state change
        'after': 'notify_deployment',    # Runs after state change
    }
]
```

### Conditions and Guards: `conditions`, `unless`

Conditions are callbacks that must return `True` for the transition to proceed. If any condition returns `False`, the transition is blocked.

```python
class Order:
    def __init__(self):
        self.paid = False
        self.items_in_stock = True

    def is_paid(self):
        return self.paid

    def is_in_stock(self):
        return self.items_in_stock

    def is_fraud(self):
        return False

transitions = [
    {
        'trigger': 'ship',
        'source': 'confirmed',
        'dest': 'shipped',
        'conditions': ['is_paid', 'is_in_stock'],  # ALL must return True
        'unless': 'is_fraud',                        # Must return False
    }
]
```

`conditions` -- a list of callbacks that **all** must return `True`.
`unless` -- a list of callbacks that **all** must return `False` (logical inverse of conditions).

When a transition with conditions is blocked, the trigger method returns `False` (rather than raising an error). Multiple transitions with the same trigger but different conditions are evaluated in order; the first one whose conditions pass is executed.

### Global Callbacks on the Machine

```python
machine = Machine(
    model=obj,
    states=states,
    transitions=transitions,
    initial='idle',
    before_state_change='log_transition_start',   # Runs before every state change
    after_state_change='log_transition_complete',  # Runs after every state change
    prepare_event='setup_context',                 # Runs before any transition evaluation
    finalize_event='cleanup',                      # Runs after every transition (always)
    on_exception='handle_error',                   # Runs if any callback raises
    on_final='mark_complete',                      # Runs when entering a final state
)
```

The `finalize_event` callback is especially useful -- it runs regardless of whether the transition succeeded, failed a condition, or raised an exception. It acts like a `finally` block.

### Passing Arguments to Callbacks

When `send_event=False` (default), trigger arguments are forwarded to callbacks as `*args, **kwargs`:

```python
class PaymentProcessor:
    def before_charge(self, amount, currency='USD'):
        print(f"Charging {amount} {currency}")

# Trigger with arguments:
processor.charge(100, currency='EUR')
# Calls before_charge(100, currency='EUR')
```

When `send_event=True`, callbacks receive a single `EventData` object:

```python
class PaymentProcessor:
    def before_charge(self, event):
        amount = event.kwargs.get('amount')
        print(f"Charging {amount}")

processor.charge(amount=100)
```

---

## Ordered Transitions and Special Values

### Special Source and Destination Values

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

### Multiple Source States

```python
# A single trigger can have multiple source states
{'trigger': 'fail', 'source': ['running', 'installing', 'degraded'], 'dest': 'failed'}
```

### Multiple Transitions with the Same Trigger

When multiple transitions share the same trigger name, they are evaluated in the order they were defined. The first transition whose conditions pass is executed; the rest are skipped.

```python
transitions = [
    # Evaluated first: requires is_premium
    {'trigger': 'ship', 'source': 'paid', 'dest': 'express_shipping',
     'conditions': 'is_premium'},
    # Evaluated second: fallback for non-premium
    {'trigger': 'ship', 'source': 'paid', 'dest': 'standard_shipping'},
]
```

### Reflexive vs. Internal Transitions

A reflexive transition (`dest='='`) exits and re-enters the current state, firing `on_exit` and `on_enter`. An internal transition (`dest=None`) does not change state and does **not** fire `on_exit` or `on_enter`.

```python
# Reflexive: on_exit and on_enter fire
{'trigger': 'refresh', 'source': 'active', 'dest': '='}

# Internal: only before/after fire, state stays the same
{'trigger': 'heartbeat', 'source': 'active', 'dest': None}
```
