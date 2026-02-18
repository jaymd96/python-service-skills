# punq — API Reference

> Part of the punq skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Core API](#core-api)
  - [Creating a Container](#creating-a-container)
  - [Registering Services](#registering-services)
  - [Resolving Services](#resolving-services)
  - [Singleton vs. Transient Scope](#singleton-vs-transient-scope)
- [API Quick Reference](#api-quick-reference)

## Core API

### Creating a Container

```python
import punq

container = punq.Container()
```

The `Container` is the central object. You register services into it, then resolve
them out of it. A typical application creates one container at startup and uses it
throughout the application lifecycle.

### Registering Services

The `register` method is the primary way to tell the container how to create objects.
It has several modes of operation.

#### Signature

```python
container.register(
    service,           # The abstract type or interface (usually an ABC or Protocol)
    implementation=None,  # The concrete class, factory, or instance
    instance=None,     # A pre-built instance to always return (singleton)
    factory=None,      # A callable that creates the service
    scope=punq.Scope.dependent,  # Lifecycle scope
)
```

#### Registration by Type (Auto-Wiring)

When you register a concrete class, punq inspects its `__init__` type annotations
to automatically resolve and inject dependencies.

```python
import punq

class UserRepository:
    pass

class UserService:
    def __init__(self, repo: UserRepository):
        self.repo = repo

container = punq.Container()
container.register(UserRepository)
container.register(UserService)

service = container.resolve(UserService)
# punq auto-creates a UserRepository and injects it into UserService
assert isinstance(service.repo, UserRepository)
```

#### Registration with an Interface/Base Class

You can register a concrete type against an abstract base class or protocol.
When resolving, you ask for the abstract type and get the concrete implementation.

```python
import abc
import punq

class MessageSender(abc.ABC):
    @abc.abstractmethod
    def send(self, message: str) -> None:
        pass

class EmailSender(MessageSender):
    def send(self, message: str) -> None:
        print(f"Sending email: {message}")

container = punq.Container()
container.register(MessageSender, EmailSender)

sender = container.resolve(MessageSender)
assert isinstance(sender, EmailSender)
sender.send("Hello!")  # "Sending email: Hello!"
```

#### Registration with a Factory Function

A factory function gives you full control over how an object is created.
punq will auto-inject the factory's annotated parameters.

```python
import punq

class DatabaseConfig:
    def __init__(self):
        self.connection_string = "postgresql://localhost/mydb"

class Database:
    def __init__(self, connection_string: str):
        self.connection_string = connection_string

def create_database(config: DatabaseConfig) -> Database:
    return Database(connection_string=config.connection_string)

container = punq.Container()
container.register(DatabaseConfig)
container.register(Database, factory=create_database)

db = container.resolve(Database)
assert db.connection_string == "postgresql://localhost/mydb"
```

#### Registration with a Specific Instance

If you already have an object you want the container to return every time,
use the `instance` parameter.

```python
import punq

class AppConfig:
    def __init__(self, debug: bool, version: str):
        self.debug = debug
        self.version = version

config = AppConfig(debug=True, version="1.0.0")

container = punq.Container()
container.register(AppConfig, instance=config)

resolved = container.resolve(AppConfig)
assert resolved is config  # Same object
assert resolved.debug is True
```

#### Passing Explicit Constructor Arguments

You can pass keyword arguments to `register` that will be forwarded to the
constructor or factory when the service is resolved.

```python
import punq

class GreetingService:
    def __init__(self, greeting: str, loud: bool):
        self.greeting = greeting
        self.loud = loud

    def greet(self, name: str) -> str:
        msg = f"{self.greeting}, {name}!"
        return msg.upper() if self.loud else msg

container = punq.Container()
container.register(GreetingService, greeting="Hello", loud=True)

svc = container.resolve(GreetingService)
assert svc.greet("World") == "HELLO, WORLD!"
```

### Resolving Services

The `resolve` method looks up a registered service type and returns an instance.

```python
service = container.resolve(ServiceType)
```

If the service depends on other registered types (via type annotations in `__init__`),
those dependencies are resolved recursively.

```python
import punq

class Logger:
    def log(self, msg: str) -> None:
        print(msg)

class Repository:
    def __init__(self, logger: Logger):
        self.logger = logger

class Service:
    def __init__(self, repo: Repository, logger: Logger):
        self.repo = repo
        self.logger = logger

container = punq.Container()
container.register(Logger)
container.register(Repository)
container.register(Service)

# Resolves the full dependency tree: Logger -> Repository -> Service
service = container.resolve(Service)
assert isinstance(service.repo, Repository)
assert isinstance(service.logger, Logger)
assert isinstance(service.repo.logger, Logger)
```

### Singleton vs. Transient Scope

punq supports two lifecycle scopes via the `punq.Scope` enum:

| Scope | Behavior |
|-------|----------|
| `Scope.dependent` | **Default.** A new instance is created every time `resolve` is called. (Transient) |
| `Scope.singleton` | One instance is created on first resolve and reused for all subsequent resolves. |

```python
import punq

class TransientService:
    pass

class SingletonService:
    pass

container = punq.Container()
container.register(TransientService, scope=punq.Scope.dependent)
container.register(SingletonService, scope=punq.Scope.singleton)

# Transient: different instances each time
t1 = container.resolve(TransientService)
t2 = container.resolve(TransientService)
assert t1 is not t2

# Singleton: same instance every time
s1 = container.resolve(SingletonService)
s2 = container.resolve(SingletonService)
assert s1 is s2
```

**Note:** Registering with `instance=` always behaves as a singleton regardless of
the `scope` parameter, since there is only one pre-built instance to return.

## API Quick Reference

```
# punq API Reference
# Version: 0.7.0 | Python: 3.7+

## Container

# Create a container
container = punq.Container()

## Registering Services

# Register a concrete type (auto-wiring)
container.register(ConcreteClass)

# Register interface -> implementation
container.register(AbstractBase, ConcreteImpl)

# Register with a factory function
container.register(ServiceType, factory=my_factory_func)

# Register a pre-built instance (always singleton)
container.register(ServiceType, instance=my_instance)

# Register as singleton
container.register(ServiceType, ConcreteImpl, scope=punq.Scope.singleton)

# Register as transient (default)
container.register(ServiceType, ConcreteImpl, scope=punq.Scope.dependent)

# Register with explicit constructor kwargs
container.register(ServiceType, kwarg1="value1", kwarg2=42)

## Resolving Services

# Resolve a service (creates or retrieves instance)
instance = container.resolve(ServiceType)

## Scopes

punq.Scope.dependent   # New instance per resolve (transient, default)
punq.Scope.singleton   # One instance, reused forever

## Exceptions

punq.MissingDependencyError    # Raised when a dependency cannot be resolved
punq.InvalidRegistrationError  # Raised when registration is invalid
```
