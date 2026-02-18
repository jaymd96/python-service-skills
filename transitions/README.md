# transitions

> Lightweight, object-oriented finite state machines for Python.

`transitions` is a declarative state machine library that lets you define states, transitions, callbacks, and conditions governing how an object moves through its lifecycle. It supports nested/hierarchical states, async operations, thread safety, diagram generation, and queued event processing -- all with a clean, extensible API.

| Detail | Value |
|--------|-------|
| **Package** | `transitions` |
| **Latest stable version** | 0.9.3 (verify at PyPI for newer releases) |
| **Python support** | 3.8+ |
| **License** | MIT |
| **Repository** | [github.com/pytransitions/transitions](https://github.com/pytransitions/transitions) |
| **PyPI** | [pypi.org/project/transitions](https://pypi.org/project/transitions/) |
| **Dependencies** | `six` (runtime); `pygraphviz` or `graphviz` for diagram extensions |

---

## Table of Contents

1. [Installation](#installation)
2. [Core Concepts](#core-concepts)
3. [Core API](#core-api)
4. [Callbacks](#callbacks)
5. [Nested / Hierarchical States](#nested--hierarchical-states)
6. [Diagrams](#diagrams)
7. [Extensions](#extensions)
8. [Queued Transitions](#queued-transitions)
9. [Dynamic Transitions](#dynamic-transitions)
10. [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
11. [Complete Code Examples](#complete-code-examples)

---

## Installation

```bash
# Standard installation
pip install transitions

# With diagram support (requires graphviz system package)
pip install "transitions[diagrams]"
```

If you need diagram generation, you also need the `graphviz` system package:

```bash
# macOS
brew install graphviz

# Debian / Ubuntu
sudo apt-get install graphviz

# Fedora / RHEL
sudo dnf install graphviz
```

---

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

## Nested / Hierarchical States

The `HierarchicalMachine` extension supports nested (hierarchical) state machines (HSMs). Child states are scoped within parent states, and transitions can target any level of the hierarchy.

```python
from transitions.extensions import HierarchicalMachine

states = [
    'standing',
    {
        'name': 'walking',
        'children': [
            'slow',
            'fast',
        ],
        'initial': 'slow',  # Default child state when entering 'walking'
    },
    {
        'name': 'running',
        'children': ['jogging', 'sprinting'],
        'initial': 'jogging',
    },
]

transitions = [
    {'trigger': 'walk', 'source': 'standing', 'dest': 'walking'},
    {'trigger': 'run', 'source': 'walking', 'dest': 'running'},
    {'trigger': 'stop', 'source': '*', 'dest': 'standing'},
    {'trigger': 'speed_up', 'source': 'walking_slow', 'dest': 'walking_fast'},
    {'trigger': 'sprint', 'source': 'running_jogging', 'dest': 'running_sprinting'},
]

class Person:
    pass

person = Person()
machine = HierarchicalMachine(
    model=person,
    states=states,
    transitions=transitions,
    initial='standing',
)

person.walk()
person.state          # 'walking_slow' (enters the initial child state)
person.is_walking()   # True (parent state check)
person.is_walking_slow()  # True (child state check)

person.speed_up()
person.state          # 'walking_fast'
person.is_walking()   # True (still in parent 'walking')
```

### Nested State Naming

Child states are referenced using the separator (default `_`):

- Parent: `walking`
- Child: `walking_slow`, `walking_fast`
- Deep nesting: `parent_child_grandchild`

You can change the separator:

```python
machine = HierarchicalMachine(..., separator='.')
# States: 'walking.slow', 'walking.fast'
```

### Reusing State Machines as Substates

You can reuse an entire machine definition as a child of another state:

```python
from transitions.extensions import HierarchicalMachine

# Define a sub-machine
cooking_states = ['prepping', 'cooking', 'plating']
cooking_transitions = [
    {'trigger': 'start_cooking', 'source': 'prepping', 'dest': 'cooking'},
    {'trigger': 'plate', 'source': 'cooking', 'dest': 'plating'},
]

# Use it as a child
states = [
    'idle',
    {
        'name': 'making_dinner',
        'children': cooking_states,
        'transitions': cooking_transitions,
        'initial': 'prepping',
    },
    'eating',
]
```

---

## Diagrams

The `GraphMachine` extension generates state machine diagrams using Graphviz. It is useful for documentation, debugging, and visualizing complex machines.

```python
from transitions.extensions import GraphMachine

class Matter:
    pass

model = Matter()

machine = GraphMachine(
    model=model,
    states=['solid', 'liquid', 'gas', 'plasma'],
    transitions=[
        {'trigger': 'melt', 'source': 'solid', 'dest': 'liquid'},
        {'trigger': 'evaporate', 'source': 'liquid', 'dest': 'gas'},
        {'trigger': 'ionize', 'source': 'gas', 'dest': 'plasma'},
        {'trigger': 'freeze', 'source': 'liquid', 'dest': 'solid'},
        {'trigger': 'condense', 'source': 'gas', 'dest': 'liquid'},
    ],
    initial='solid',
    show_conditions=True,         # Display condition names on edges
    show_state_attributes=True,   # Display state tags and callbacks
    title='Matter Phase Transitions',
)

# Generate image files
model.get_graph().draw('matter_states.png', prog='dot')
model.get_graph().draw('matter_states.svg', prog='dot', format='svg')

# Get the raw DOT source
dot_source = model.get_graph().string()
```

### Highlighting Current State

`GraphMachine` automatically highlights the current state in the diagram. As the model transitions, regenerating the graph shows the new active state.

```python
model.melt()
model.get_graph().draw('matter_liquid.png', prog='dot')  # 'liquid' is highlighted
```

---

## Extensions

The `transitions` package provides several extensions that can be combined using mixins.

### `HierarchicalMachine` -- Nested/Hierarchical States

```python
from transitions.extensions import HierarchicalMachine
```

Supports child states, initial child states, nested state naming, and parent-level `is_<parent>()` checks. See the [Nested / Hierarchical States](#nested--hierarchical-states) section above.

### `GraphMachine` -- Diagram Generation

```python
from transitions.extensions import GraphMachine
```

Generates Graphviz diagrams of the state machine. Requires `pygraphviz` or `graphviz` to be installed. See the [Diagrams](#diagrams) section above.

### `AsyncMachine` -- Async Support

```python
from transitions.extensions.asyncio import AsyncMachine
```

`AsyncMachine` makes all trigger methods into coroutines and supports `async` callbacks. This is essential when your state machine callbacks perform async I/O.

```python
import asyncio
from transitions.extensions.asyncio import AsyncMachine

class AsyncOrder:
    async def on_enter_processing(self):
        print("Processing order...")
        await asyncio.sleep(1)  # Simulate async work
        print("Order processed.")

    async def validate_payment(self):
        await asyncio.sleep(0.5)
        return True

order = AsyncOrder()

machine = AsyncMachine(
    model=order,
    states=['pending', 'processing', 'shipped', 'delivered'],
    transitions=[
        {'trigger': 'process', 'source': 'pending', 'dest': 'processing',
         'conditions': 'validate_payment'},
        {'trigger': 'ship', 'source': 'processing', 'dest': 'shipped'},
        {'trigger': 'deliver', 'source': 'shipped', 'dest': 'delivered'},
    ],
    initial='pending',
    queued=True,  # Recommended for async to avoid race conditions
)

async def main():
    await order.process()   # Triggers are now coroutines
    print(order.state)      # 'processing'
    await order.ship()
    print(order.state)      # 'shipped'

asyncio.run(main())
```

**Key differences from `Machine`:**

- All trigger methods are coroutines (`await model.trigger_name()`).
- All callbacks can be `async def` coroutines.
- Condition callbacks can be async.
- Use `queued=True` to prevent concurrent transitions from interleaving.

### `LockedMachine` -- Thread-Safe Transitions

```python
from transitions.extensions import LockedMachine
```

`LockedMachine` wraps all trigger methods with a reentrant lock (`threading.RLock`), making the state machine safe for use from multiple threads.

```python
from transitions.extensions import LockedMachine
import threading

class SharedResource:
    pass

resource = SharedResource()

machine = LockedMachine(
    model=resource,
    states=['idle', 'busy', 'error'],
    transitions=[
        {'trigger': 'start', 'source': 'idle', 'dest': 'busy'},
        {'trigger': 'finish', 'source': 'busy', 'dest': 'idle'},
        {'trigger': 'fail', 'source': 'busy', 'dest': 'error'},
        {'trigger': 'reset', 'source': 'error', 'dest': 'idle'},
    ],
    initial='idle',
)

# Safe to call from multiple threads:
def worker():
    resource.start()
    # ... do work ...
    resource.finish()

threads = [threading.Thread(target=worker) for _ in range(5)]
for t in threads:
    t.start()
```

### Combining Extensions (Mixins)

Extensions are implemented as mixins and can be combined:

```python
from transitions.extensions import HierarchicalGraphMachine
# Combines HierarchicalMachine + GraphMachine

from transitions.extensions import LockedHierarchicalMachine
# Combines LockedMachine + HierarchicalMachine

from transitions.extensions.asyncio import AsyncGraphMachine
# Combines AsyncMachine + GraphMachine
```

---

## Queued Transitions

When `queued=True`, events triggered during a callback are not processed immediately. Instead, they are placed in a queue and processed sequentially after the current transition completes.

```python
class TrafficLight:
    def on_enter_yellow(self):
        # Without queued=True, this would cause a re-entrant transition
        # and potentially corrupt state. With queued=True, it is safe.
        self.to_red()

machine = Machine(
    model=light,
    states=['red', 'yellow', 'green'],
    transitions=[
        {'trigger': 'next', 'source': 'green', 'dest': 'yellow'},
        {'trigger': 'next', 'source': 'yellow', 'dest': 'red'},
        {'trigger': 'next', 'source': 'red', 'dest': 'green'},
    ],
    initial='red',
    queued=True,  # Critical for re-entrant callbacks
)
```

**When to use `queued=True`:**

- When callbacks may trigger additional transitions (re-entrant transitions).
- With `AsyncMachine` to prevent interleaved async transitions.
- When you want deterministic, sequential event processing.

**Without `queued=True`:** A transition triggered inside a callback executes immediately (nested), which can lead to confusing callback ordering and subtle bugs.

---

## Dynamic Transitions

States and transitions can be added or removed at runtime, allowing the state machine to evolve dynamically.

### Adding Transitions at Runtime

```python
from transitions import Machine

class Robot:
    pass

robot = Robot()
machine = Machine(model=robot, states=['idle', 'working'], initial='idle')

# Add a new state
machine.add_states(['charging', 'maintenance'])

# Add new transitions
machine.add_transition(
    trigger='start_work',
    source='idle',
    dest='working',
    before='log_start',
)

machine.add_transition(
    trigger='charge',
    source='idle',
    dest='charging',
    conditions='battery_low',
)

# Add a transition with multiple sources
machine.add_transition(
    trigger='emergency_stop',
    source='*',
    dest='idle',
)
```

### Removing Transitions

```python
# Remove a specific trigger entirely
machine.remove_transition(trigger='charge', source='idle', dest='charging')
```

### Replacing the Entire Transition Set

For more dramatic runtime changes, you can rebuild the machine's transition table by clearing and re-adding:

```python
# Remove all transitions for a trigger
machine.remove_transition(trigger='start_work', source='idle', dest='working')

# Add a replacement
machine.add_transition(
    trigger='start_work',
    source='idle',
    dest='working',
    conditions='has_task_assigned',
)
```

---

## Gotchas and Common Mistakes

### 1. Model vs. Machine Confusion

The **model** is the object whose state is being managed. The **machine** is the orchestrator. When `model=None`, the Machine itself is the model, which can be confusing in larger applications.

```python
# Confusing: Machine is its own model
machine = Machine(states=['a', 'b'], initial='a')
machine.state   # 'a' -- works, but mixes concerns

# Clear: separate model and machine
class MyObj:
    pass

obj = MyObj()
machine = Machine(model=obj, states=['a', 'b'], initial='a')
obj.state       # 'a'
```

**Recommendation:** Always use a separate model object in production code. Use `model=None` (Machine-as-model) only for quick prototyping or testing.

### 2. Callback Ordering Surprises

The callback execution order is fixed and sometimes counter-intuitive. The most common surprise: `on_exit` of the source state fires **before** the state actually changes, but `on_enter` of the destination fires **after**.

```
prepare_event -> prepare -> conditions -> before_state_change ->
before -> on_exit -> STATE CHANGES -> on_enter -> after ->
on_final -> after_state_change -> finalize_event
```

If you need to know the destination state inside an `on_exit` callback, you must inspect the transition object via `EventData` (requires `send_event=True`):

```python
machine = Machine(..., send_event=True)

def on_exit_running(self, event):
    dest = event.transition.dest
    print(f"Leaving 'running', heading to '{dest}'")
```

### 3. Auto-Transitions and Naming Conflicts

With `auto_transitions=True` (the default), the machine generates `to_<state>()` methods for every state. If your model already has a method with a conflicting name, it will be silently overwritten.

```python
class MyModel:
    def to_active(self):
        """My custom method."""
        pass

# WARNING: Machine will overwrite to_active() with the auto-transition!
machine = Machine(model=obj, states=['idle', 'active'], initial='idle')
```

**Solutions:**

- Set `auto_transitions=False` if you do not need `to_<state>()` methods.
- Set `model_override=True` to only override methods already present on the model.
- Avoid state names that conflict with your model's methods.

### 4. Thread Safety Without LockedMachine

The standard `Machine` is **not** thread-safe. If multiple threads trigger transitions on the same model, you can get corrupted state, skipped callbacks, or race conditions.

```python
# WRONG: standard Machine in multi-threaded context
machine = Machine(model=shared_obj, ...)

# CORRECT: use LockedMachine for thread safety
from transitions.extensions import LockedMachine
machine = LockedMachine(model=shared_obj, ...)
```

### 5. State Comparison (String vs. Object vs. Enum)

How you compare state depends on how states were defined:

```python
# String states
model.state == 'running'        # True
model.state == 'Running'        # False -- case-sensitive!

# Enum states
model.state == TrafficState.RED  # True
model.state == 'RED'             # False -- enum member, not string

# Always safe: use the auto-generated is_<state>() methods
model.is_running()              # Works regardless of state type
```

### 6. Triggers Returning True/False vs. Raising Exceptions

By default, calling a trigger that is not valid for the current state raises a `MachineError`. If you set `ignore_invalid_triggers=True`, invalid triggers silently return `False` instead.

```python
# Default behavior: raises MachineError
order.ship()  # MachineError: "Can't trigger event 'ship' from state 'pending'"

# With ignore_invalid_triggers=True: returns False
machine = Machine(..., ignore_invalid_triggers=True)
order.ship()  # Returns False silently
```

**Tip:** Use `model.may_<trigger>()` to check before triggering:

```python
if order.may_ship():
    order.ship()
else:
    print("Cannot ship yet")
```

### 7. Conditions Block Silently

When a condition returns `False`, the transition simply does not happen. The trigger method returns `False` but no exception is raised. This can make debugging difficult.

```python
# The transition silently fails if is_paid() returns False
result = order.ship()
if not result:
    print("Transition blocked by conditions")
```

### 8. `send_event` Affects All Callbacks

The `send_event=True` setting is machine-wide. Once set, **all** callbacks (conditions, before, after, on_enter, on_exit) receive `EventData` instead of raw arguments. You cannot mix the two styles in the same machine.

### 9. Multiple Transitions with the Same Trigger

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

### 10. Reflexive vs. Internal Transitions

A reflexive transition (`dest='='`) exits and re-enters the current state, firing `on_exit` and `on_enter`. An internal transition (`dest=None`) does not change state and does **not** fire `on_exit` or `on_enter`.

```python
# Reflexive: on_exit and on_enter fire
{'trigger': 'refresh', 'source': 'active', 'dest': '='}

# Internal: only before/after fire, state stays the same
{'trigger': 'heartbeat', 'source': 'active', 'dest': None}
```

---

## Complete Code Examples

### Example 1: Basic FSM -- Traffic Light

A simple traffic light cycling through states with timed transitions.

```python
from transitions import Machine


class TrafficLight:
    """A simple traffic light FSM."""

    states = ['green', 'yellow', 'red']

    transitions = [
        {'trigger': 'slow_down', 'source': 'green', 'dest': 'yellow',
         'after': 'log_change'},
        {'trigger': 'stop', 'source': 'yellow', 'dest': 'red',
         'after': 'log_change'},
        {'trigger': 'go', 'source': 'red', 'dest': 'green',
         'after': 'log_change'},
    ]

    def __init__(self):
        self.machine = Machine(
            model=self,
            states=self.states,
            transitions=self.transitions,
            initial='red',
        )

    def log_change(self):
        print(f"Traffic light is now {self.state.upper()}")


# Usage
light = TrafficLight()
print(f"Initial: {light.state}")   # red

light.go()                          # Traffic light is now GREEN
light.slow_down()                   # Traffic light is now YELLOW
light.stop()                        # Traffic light is now RED

# Check state
print(light.is_red())               # True
print(light.may_go())               # True
print(light.may_slow_down())        # False (not green)
```

### Example 2: Order Lifecycle with Conditions and Guards

An e-commerce order with payment validation, stock checks, and fraud detection.

```python
from transitions import Machine, State
from datetime import datetime


class Order:
    """E-commerce order with state machine lifecycle."""

    states = [
        State(name='draft'),
        State(name='submitted', on_enter=['record_submission_time']),
        State(name='confirmed', on_enter=['send_confirmation_email']),
        State(name='shipped', on_enter=['generate_tracking_number']),
        State(name='delivered', on_enter=['request_review'], final=True),
        State(name='cancelled', on_enter=['process_refund'], final=True),
    ]

    transitions = [
        {
            'trigger': 'submit',
            'source': 'draft',
            'dest': 'submitted',
            'conditions': ['has_items', 'has_shipping_address'],
        },
        {
            'trigger': 'confirm',
            'source': 'submitted',
            'dest': 'confirmed',
            'conditions': ['payment_verified'],
            'unless': ['is_fraudulent'],
            'before': 'charge_payment',
        },
        {
            'trigger': 'ship',
            'source': 'confirmed',
            'dest': 'shipped',
            'conditions': ['items_in_stock'],
            'after': 'notify_customer',
        },
        {
            'trigger': 'deliver',
            'source': 'shipped',
            'dest': 'delivered',
        },
        {
            'trigger': 'cancel',
            'source': ['draft', 'submitted', 'confirmed'],
            'dest': 'cancelled',
            'before': 'record_cancellation_reason',
        },
    ]

    def __init__(self, order_id, items=None):
        self.order_id = order_id
        self.items = items or []
        self.shipping_address = None
        self.payment_method = None
        self.submitted_at = None
        self.tracking_number = None
        self.cancellation_reason = None

        self.machine = Machine(
            model=self,
            states=self.states,
            transitions=self.transitions,
            initial='draft',
            send_event=True,
            finalize_event='log_transition',
        )

    # -- Conditions --
    def has_items(self, event):
        return len(self.items) > 0

    def has_shipping_address(self, event):
        return self.shipping_address is not None

    def payment_verified(self, event):
        return self.payment_method is not None

    def is_fraudulent(self, event):
        # Placeholder fraud detection
        return False

    def items_in_stock(self, event):
        # Placeholder stock check
        return True

    # -- Callbacks --
    def record_submission_time(self, event):
        self.submitted_at = datetime.utcnow()
        print(f"Order {self.order_id} submitted at {self.submitted_at}")

    def send_confirmation_email(self, event):
        print(f"Order {self.order_id}: Confirmation email sent")

    def charge_payment(self, event):
        print(f"Order {self.order_id}: Payment charged via {self.payment_method}")

    def generate_tracking_number(self, event):
        self.tracking_number = f"TRK-{self.order_id}-001"
        print(f"Order {self.order_id}: Tracking number {self.tracking_number}")

    def notify_customer(self, event):
        print(f"Order {self.order_id}: Customer notified of shipment")

    def request_review(self, event):
        print(f"Order {self.order_id}: Review request sent")

    def process_refund(self, event):
        print(f"Order {self.order_id}: Refund processed")

    def record_cancellation_reason(self, event):
        self.cancellation_reason = event.kwargs.get('reason', 'No reason given')
        print(f"Order {self.order_id}: Cancelled - {self.cancellation_reason}")

    def log_transition(self, event):
        print(f"  [{self.order_id}] {event.transition.source} -> {self.state}")


# Usage
order = Order('ORD-100', items=['Widget', 'Gadget'])
order.shipping_address = '123 Main St'
order.payment_method = 'credit_card'

print(f"State: {order.state}")      # draft

order.submit()                       # submitted
order.confirm()                      # confirmed (payment charged)
order.ship()                         # shipped (tracking number generated)
order.deliver()                      # delivered (review requested)

print(f"Final state: {order.state}") # delivered
print(f"Tracking: {order.tracking_number}")
```

### Example 3: Hierarchical States

A character controller with nested movement states.

```python
from transitions.extensions import HierarchicalMachine


class Character:
    """Game character with hierarchical movement states."""

    states = [
        'idle',
        {
            'name': 'moving',
            'children': [
                {
                    'name': 'walking',
                    'children': ['forward', 'backward'],
                    'initial': 'forward',
                },
                'running',
                'crouching',
            ],
            'initial': 'walking',
        },
        {
            'name': 'airborne',
            'children': ['jumping', 'falling'],
            'initial': 'jumping',
        },
        'dead',
    ]

    transitions = [
        {'trigger': 'move', 'source': 'idle', 'dest': 'moving'},
        {'trigger': 'stop', 'source': 'moving', 'dest': 'idle'},
        {'trigger': 'sprint', 'source': 'moving_walking', 'dest': 'moving_running'},
        {'trigger': 'crouch', 'source': 'moving_walking', 'dest': 'moving_crouching'},
        {'trigger': 'jump', 'source': ['idle', 'moving'], 'dest': 'airborne'},
        {'trigger': 'land', 'source': 'airborne', 'dest': 'idle'},
        {'trigger': 'die', 'source': '*', 'dest': 'dead'},
        {'trigger': 'walk_backward', 'source': 'moving_walking_forward',
         'dest': 'moving_walking_backward'},
    ]

    def __init__(self):
        self.machine = HierarchicalMachine(
            model=self,
            states=self.states,
            transitions=self.transitions,
            initial='idle',
        )


# Usage
char = Character()
print(char.state)                    # idle

char.move()
print(char.state)                    # moving_walking_forward

print(char.is_moving())             # True (parent check)
print(char.is_moving_walking())     # True (intermediate check)
print(char.is_idle())               # False

char.sprint()
print(char.state)                    # moving_running

char.jump()
print(char.state)                    # airborne_jumping

char.land()
print(char.state)                    # idle
```

### Example 4: Async State Machine

An async pipeline for processing jobs with I/O-bound operations.

```python
import asyncio
from transitions.extensions.asyncio import AsyncMachine


class AsyncJobProcessor:
    """Async job processor with state machine."""

    states = ['idle', 'fetching', 'processing', 'saving', 'done', 'error']

    transitions = [
        {
            'trigger': 'start',
            'source': 'idle',
            'dest': 'fetching',
            'before': 'validate_job',
        },
        {
            'trigger': 'process',
            'source': 'fetching',
            'dest': 'processing',
        },
        {
            'trigger': 'save',
            'source': 'processing',
            'dest': 'saving',
        },
        {
            'trigger': 'complete',
            'source': 'saving',
            'dest': 'done',
            'after': 'cleanup',
        },
        {
            'trigger': 'fail',
            'source': '*',
            'dest': 'error',
            'before': 'capture_error',
        },
    ]

    def __init__(self, job_id: str):
        self.job_id = job_id
        self.data = None
        self.result = None
        self.error_message = None

        self.machine = AsyncMachine(
            model=self,
            states=self.states,
            transitions=self.transitions,
            initial='idle',
            queued=True,
        )

    async def validate_job(self):
        print(f"[{self.job_id}] Validating...")
        await asyncio.sleep(0.1)

    async def on_enter_fetching(self):
        print(f"[{self.job_id}] Fetching data...")
        await asyncio.sleep(0.5)  # Simulate network I/O
        self.data = {"payload": "fetched_data"}

    async def on_enter_processing(self):
        print(f"[{self.job_id}] Processing data...")
        await asyncio.sleep(0.3)  # Simulate CPU work
        self.result = f"processed_{self.data['payload']}"

    async def on_enter_saving(self):
        print(f"[{self.job_id}] Saving result...")
        await asyncio.sleep(0.2)  # Simulate database write

    async def cleanup(self):
        print(f"[{self.job_id}] Cleanup complete.")

    async def capture_error(self, error_msg="Unknown error"):
        self.error_message = error_msg
        print(f"[{self.job_id}] Error: {error_msg}")

    async def run(self) -> bool:
        """Execute the full job pipeline."""
        try:
            await self.start()
            await self.process()
            await self.save()
            await self.complete()
            return True
        except Exception as e:
            await self.fail(error_msg=str(e))
            return False


async def main():
    job = AsyncJobProcessor("JOB-001")
    success = await job.run()
    print(f"Job finished: state={job.state}, success={success}")
    print(f"Result: {job.result}")


asyncio.run(main())
```

### Example 5: Integration with Blinker for Event Broadcasting

Using the `blinker` library to broadcast state changes to decoupled listeners.

```python
from transitions import Machine, State
from blinker import signal

# Define signals for state change events
state_entered = signal('state-entered')
state_exited = signal('state-exited')
transition_complete = signal('transition-complete')


class ObservableStateMixin:
    """Mixin that broadcasts state machine events via blinker signals."""

    def _broadcast_after_state_change(self, **kwargs):
        transition_complete.send(
            self,
            model=self,
            new_state=self.state,
        )


class Document(ObservableStateMixin):
    """Document with an observable state machine lifecycle."""

    states = [
        State(name='draft'),
        State(name='review'),
        State(name='approved'),
        State(name='published', final=True),
        State(name='rejected'),
    ]

    transitions = [
        {'trigger': 'submit_for_review', 'source': 'draft', 'dest': 'review'},
        {'trigger': 'approve', 'source': 'review', 'dest': 'approved'},
        {'trigger': 'reject', 'source': 'review', 'dest': 'rejected',
         'before': 'record_rejection'},
        {'trigger': 'publish', 'source': 'approved', 'dest': 'published'},
        {'trigger': 'revise', 'source': 'rejected', 'dest': 'draft'},
    ]

    def __init__(self, title: str):
        self.title = title
        self.rejection_reason = None

        self.machine = Machine(
            model=self,
            states=self.states,
            transitions=self.transitions,
            initial='draft',
            after_state_change='_broadcast_after_state_change',
        )

    def record_rejection(self, reason='Not specified'):
        self.rejection_reason = reason


# -- Listeners (completely decoupled from the Document class) --

def audit_logger(sender, **kwargs):
    """Log all state transitions for audit trail."""
    print(f"[AUDIT] {sender.title}: entered '{kwargs['new_state']}'")


def notification_service(sender, **kwargs):
    """Send notifications on specific state changes."""
    new_state = kwargs['new_state']
    if new_state == 'review':
        print(f"[NOTIFY] '{sender.title}' is ready for review")
    elif new_state == 'published':
        print(f"[NOTIFY] '{sender.title}' has been published!")
    elif new_state == 'rejected':
        print(f"[NOTIFY] '{sender.title}' was rejected: {sender.rejection_reason}")


# Connect listeners to the signal
transition_complete.connect(audit_logger)
transition_complete.connect(notification_service)


# Usage
doc = Document("Q1 Report")

doc.submit_for_review()
# [AUDIT] Q1 Report: entered 'review'
# [NOTIFY] 'Q1 Report' is ready for review

doc.reject(reason="Missing financial data")
# [AUDIT] Q1 Report: entered 'rejected'
# [NOTIFY] 'Q1 Report' was rejected: Missing financial data

doc.revise()
# [AUDIT] Q1 Report: entered 'draft'

doc.submit_for_review()
# [AUDIT] Q1 Report: entered 'review'
# [NOTIFY] 'Q1 Report' is ready for review

doc.approve()
# [AUDIT] Q1 Report: entered 'approved'

doc.publish()
# [AUDIT] Q1 Report: entered 'published'
# [NOTIFY] 'Q1 Report' has been published!
```

### Example 6: Complete Machine with Error Handling and Finalization

A robust state machine demonstrating global error handling, finalization, and the `may_` check pattern.

```python
from transitions import Machine, State, MachineError
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


class DeploymentPipeline:
    """CI/CD deployment pipeline with comprehensive error handling."""

    states = [
        State(name='idle'),
        State(name='building', on_enter=['start_build_timer']),
        State(name='testing', on_enter=['start_test_suite']),
        State(name='staging', on_enter=['deploy_to_staging']),
        State(name='production', on_enter=['deploy_to_production'], final=True),
        State(name='failed', on_enter=['alert_team']),
        State(name='rolled_back', on_enter=['confirm_rollback']),
    ]

    transitions = [
        {'trigger': 'build', 'source': 'idle', 'dest': 'building'},
        {'trigger': 'test', 'source': 'building', 'dest': 'testing',
         'conditions': 'build_succeeded'},
        {'trigger': 'stage', 'source': 'testing', 'dest': 'staging',
         'conditions': 'all_tests_passed'},
        {'trigger': 'deploy', 'source': 'staging', 'dest': 'production',
         'conditions': ['staging_healthy', 'deploy_window_open'],
         'unless': 'freeze_active'},
        {'trigger': 'fail', 'source': ['building', 'testing', 'staging'],
         'dest': 'failed'},
        {'trigger': 'rollback', 'source': ['staging', 'failed'],
         'dest': 'rolled_back',
         'conditions': 'has_previous_version'},
        {'trigger': 'reset', 'source': ['failed', 'rolled_back'],
         'dest': 'idle'},
    ]

    def __init__(self, app_name: str):
        self.app_name = app_name
        self.build_success = False
        self.test_results = None
        self.error_details = None
        self._previous_version = 'v1.0.0'

        self.machine = Machine(
            model=self,
            states=self.states,
            transitions=self.transitions,
            initial='idle',
            send_event=True,
            on_exception='handle_exception',
            finalize_event='finalize',
            on_final='on_deployment_complete',
        )

    # Conditions
    def build_succeeded(self, event):
        return self.build_success

    def all_tests_passed(self, event):
        return self.test_results == 'passed'

    def staging_healthy(self, event):
        return True

    def deploy_window_open(self, event):
        return True

    def freeze_active(self, event):
        return False

    def has_previous_version(self, event):
        return self._previous_version is not None

    # State entry callbacks
    def start_build_timer(self, event):
        logger.info(f"[{self.app_name}] Build started")
        self.build_success = True

    def start_test_suite(self, event):
        logger.info(f"[{self.app_name}] Running tests...")
        self.test_results = 'passed'

    def deploy_to_staging(self, event):
        logger.info(f"[{self.app_name}] Deployed to staging")

    def deploy_to_production(self, event):
        logger.info(f"[{self.app_name}] Deployed to production!")

    def alert_team(self, event):
        logger.error(f"[{self.app_name}] ALERT: Pipeline failed - {self.error_details}")

    def confirm_rollback(self, event):
        logger.info(f"[{self.app_name}] Rolled back to {self._previous_version}")

    # Global callbacks
    def handle_exception(self, event):
        self.error_details = str(event.error)
        logger.exception(f"[{self.app_name}] Exception during transition")

    def finalize(self, event):
        logger.debug(f"[{self.app_name}] Transition finalized: state={self.state}")

    def on_deployment_complete(self, event):
        logger.info(f"[{self.app_name}] === DEPLOYMENT COMPLETE ===")


# Usage
pipeline = DeploymentPipeline("my-api")

# Safe transition pattern using may_ checks
steps = ['build', 'test', 'stage', 'deploy']
for step in steps:
    checker = getattr(pipeline, f'may_{step}')
    if checker():
        trigger = getattr(pipeline, step)
        trigger()
        logger.info(f"After '{step}': state = {pipeline.state}")
    else:
        logger.warning(f"Cannot '{step}' from state '{pipeline.state}'")
        break
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

### Extensions

| Extension | Import | Purpose |
|-----------|--------|---------|
| `HierarchicalMachine` | `from transitions.extensions import HierarchicalMachine` | Nested/hierarchical states |
| `GraphMachine` | `from transitions.extensions import GraphMachine` | Diagram generation |
| `AsyncMachine` | `from transitions.extensions.asyncio import AsyncMachine` | Async support |
| `LockedMachine` | `from transitions.extensions import LockedMachine` | Thread-safe transitions |
| `HierarchicalGraphMachine` | `from transitions.extensions import HierarchicalGraphMachine` | HSM + diagrams |
| `LockedHierarchicalMachine` | `from transitions.extensions import LockedHierarchicalMachine` | HSM + thread safety |

### Callback Execution Order

```
1.  Machine.prepare_event
2.  Transition.prepare
3.  Transition.conditions / unless
4.  Machine.before_state_change
5.  Transition.before
6.  State.on_exit (source)
7.  *** STATE CHANGES ***
8.  State.on_enter (destination)
9.  Transition.after
10. Machine.on_final (if dest is final)
11. Machine.after_state_change
12. Machine.finalize_event (always runs)
```

---

## Further Reading

- [GitHub repository](https://github.com/pytransitions/transitions) -- Full README with extensive examples
- [PyPI page](https://pypi.org/project/transitions/)
- [State machine fundamentals](https://en.wikipedia.org/wiki/Finite-state_machine) -- Wikipedia overview of FSM theory
