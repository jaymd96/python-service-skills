# Blinker — Signal Patterns

> Part of the blinker skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Sender Filtering](#sender-filtering)
- [Weak References](#weak-references)
- [Anonymous Signals vs Named Signals](#anonymous-signals-vs-named-signals)

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

Sender filtering uses **identity** (`is`), not equality (`==`). The sender you connect with must be the exact same object as the sender passed to `send()`. See the [gotchas section](examples.md#3-sender-identity-uses-is-not-) for implications.

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
