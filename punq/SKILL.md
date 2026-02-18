---
name: punq
description: Minimal dependency injection container for Python (~300 lines). Use when setting up service registration and resolution, configuring singletons vs transient services, or implementing IoC patterns. Triggers on dependency injection, DI container, punq, service registration, IoC, resolve dependencies.
---

# punq — Minimal DI Container (v0.7.0)

## Quick Start

```bash
pip install punq
```

```python
import punq

container = punq.Container()
container.register(Database)
container.register(UserService)

svc = container.resolve(UserService)  # Database auto-injected
```

## Key Patterns

### Register with interface
```python
container.register(AbstractRepo, ConcreteRepo)
svc = container.resolve(AbstractRepo)  # returns ConcreteRepo instance
```

### Singleton vs transient
```python
container.register(Database, scope=punq.Scope.singleton)   # same instance always
container.register(RequestHandler, scope=punq.Scope.dependent)  # new each time (default)
```

### Factory function
```python
container.register(Database, factory=lambda: Database(url=os.getenv("DB_URL")))
```

## References

- **[api.md](references/api.md)** — Container API, register/resolve, service lifetimes, factory functions
- **[patterns.md](references/patterns.md)** — DI patterns, testing, framework integration, scoping
- **[examples.md](references/examples.md)** — Complete examples, gotchas, common mistakes
