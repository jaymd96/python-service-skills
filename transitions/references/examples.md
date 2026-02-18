# transitions — Examples & Gotchas

> Part of the transitions skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Model vs. Machine Confusion](#1-model-vs-machine-confusion)
  - [2. Callback Ordering Surprises](#2-callback-ordering-surprises)
  - [3. Auto-Transitions and Naming Conflicts](#3-auto-transitions-and-naming-conflicts)
  - [4. Thread Safety Without LockedMachine](#4-thread-safety-without-lockedmachine)
  - [5. State Comparison (String vs. Object vs. Enum)](#5-state-comparison-string-vs-object-vs-enum)
  - [6. Triggers Returning True/False vs. Raising Exceptions](#6-triggers-returning-truefalse-vs-raising-exceptions)
  - [7. Conditions Block Silently](#7-conditions-block-silently)
  - [8. send_event Affects All Callbacks](#8-send_event-affects-all-callbacks)
  - [9. Multiple Transitions with the Same Trigger](#9-multiple-transitions-with-the-same-trigger)
  - [10. Reflexive vs. Internal Transitions](#10-reflexive-vs-internal-transitions)
- [Complete Code Examples](#complete-code-examples)
  - [Example 1: Basic FSM -- Traffic Light](#example-1-basic-fsm----traffic-light)
  - [Example 2: Order Lifecycle with Conditions and Guards](#example-2-order-lifecycle-with-conditions-and-guards)
  - [Example 3: Hierarchical States](#example-3-hierarchical-states)
  - [Example 4: Async State Machine](#example-4-async-state-machine)
  - [Example 5: Integration with Blinker for Event Broadcasting](#example-5-integration-with-blinker-for-event-broadcasting)
  - [Example 6: Complete Machine with Error Handling and Finalization](#example-6-complete-machine-with-error-handling-and-finalization)
- [Further Reading](#further-reading)

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

## Further Reading

- [GitHub repository](https://github.com/pytransitions/transitions) -- Full README with extensive examples
- [PyPI page](https://pypi.org/project/transitions/)
- [State machine fundamentals](https://en.wikipedia.org/wiki/Finite-state_machine) -- Wikipedia overview of FSM theory
