# transitions — Advanced Features

> Part of the transitions skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Nested / Hierarchical States](#nested--hierarchical-states)
  - [Nested State Naming](#nested-state-naming)
  - [Reusing State Machines as Substates](#reusing-state-machines-as-substates)
- [Diagrams](#diagrams)
  - [Highlighting Current State](#highlighting-current-state)
- [Extensions](#extensions)
  - [HierarchicalMachine -- Nested/Hierarchical States](#hierarchicalmachine----nestedhierarchical-states)
  - [GraphMachine -- Diagram Generation](#graphmachine----diagram-generation)
  - [AsyncMachine -- Async Support](#asyncmachine----async-support)
  - [LockedMachine -- Thread-Safe Transitions](#lockedmachine----thread-safe-transitions)
  - [Combining Extensions (Mixins)](#combining-extensions-mixins)
- [Queued Transitions](#queued-transitions)
- [Dynamic Transitions](#dynamic-transitions)
  - [Adding Transitions at Runtime](#adding-transitions-at-runtime)
  - [Removing Transitions](#removing-transitions)
  - [Replacing the Entire Transition Set](#replacing-the-entire-transition-set)

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

### Extensions Summary

| Extension | Import | Purpose |
|-----------|--------|---------|
| `HierarchicalMachine` | `from transitions.extensions import HierarchicalMachine` | Nested/hierarchical states |
| `GraphMachine` | `from transitions.extensions import GraphMachine` | Diagram generation |
| `AsyncMachine` | `from transitions.extensions.asyncio import AsyncMachine` | Async support |
| `LockedMachine` | `from transitions.extensions import LockedMachine` | Thread-safe transitions |
| `HierarchicalGraphMachine` | `from transitions.extensions import HierarchicalGraphMachine` | HSM + diagrams |
| `LockedHierarchicalMachine` | `from transitions.extensions import LockedHierarchicalMachine` | HSM + thread safety |

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
