# pluggy — Hook Patterns

> Part of the pluggy skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Hook Specifications](#hook-specifications)
  - [Basic Hookspec](#basic-hookspec)
  - [firstresult=True -- Stop at First Non-None Result](#firstresulttrue----stop-at-first-non-none-result)
  - [historic=True -- Replay to Late-Registered Plugins](#historictrue----replay-to-late-registered-plugins)
  - [warn_on_impl -- Spec/Impl Mismatch Warning](#warn_on_impl----specimpl-mismatch-warning)
- [Hook Implementations](#hook-implementations)
  - [Basic Implementation](#basic-implementation)
  - [Ordering: tryfirst and trylast](#ordering-tryfirst-and-trylast)
  - [wrapper=True -- Wrapping Other Implementations](#wrappertrue----wrapping-other-implementations)
  - [specname -- Implementing a Differently-Named Spec](#specname----implementing-a-differently-named-spec)
  - [optionalhook=True -- Suppress Missing Spec Warning](#optionalhooktrue----suppress-missing-spec-warning)
- [Hook Wrappers (In Depth)](#hook-wrappers-in-depth)
  - [The Yield-Based Pattern](#the-yield-based-pattern)
  - [Error Handling in Wrappers](#error-handling-in-wrappers)
  - [Wrapper Ordering](#wrapper-ordering)
  - [Real-World Wrapper: Transaction Management](#real-world-wrapper-transaction-management)

## Hook Specifications

Hook specifications define the **contract** -- the function signature, arguments, and expected behavior of an extension point. They are declared in a class and registered with the plugin manager.

### Basic Hookspec

```python
import pluggy

hookspec = pluggy.HookspecMarker("myproject")

class MyProjectSpec:
    """Hook specifications for myproject."""

    @hookspec
    def process_item(self, item):
        """Process a single item.

        Called for each item in the pipeline. All registered
        implementations are called and their results collected.

        Args:
            item: The item to process.

        Returns:
            Processed item or transformation result.
        """
```

### firstresult=True -- Stop at First Non-None Result

When a hookspec is marked with `firstresult=True`, the hook caller stops iterating through implementations as soon as one returns a non-`None` value. This is useful for "lookup" or "resolution" style hooks.

```python
class MyProjectSpec:
    @hookspec(firstresult=True)
    def resolve_path(self, path):
        """Resolve a path to a resource.

        The first plugin that can resolve this path wins.
        Returns None to pass to the next implementation.
        """
```

When called:
```python
# Only the first non-None return value is returned (not a list)
result = pm.hook.resolve_path(path="/some/path")
# result is a single value, not a list
```

### historic=True -- Replay to Late-Registered Plugins

Historic hooks remember every call that was made to them. When a new plugin is registered after the hook has already been called, all previous calls are **replayed** to the new plugin's implementation. This is essential for configuration or initialization hooks.

```python
class MyProjectSpec:
    @hookspec(historic=True)
    def configure(self, config):
        """Configure a plugin with application settings.

        This is historic: if a plugin registers after configure()
        has been called, it will still receive the configuration.
        """
```

Usage pattern:
```python
pm = pluggy.PluginManager("myproject")
pm.add_hookspecs(MyProjectSpec)

# Call the historic hook
pm.hook.configure.call_historic(kwargs=dict(config=app_config))

# Later, register a new plugin -- it STILL gets the configure call
pm.register(late_plugin)
# late_plugin.configure(config=app_config) is called automatically
```

### warn_on_impl -- Spec/Impl Mismatch Warning

By default, pluggy issues a warning when:
- A hook implementation has arguments that the specification does not define
- A hook implementation is registered but no corresponding spec exists

This is controlled at the spec level and by plugin manager validation:

```python
# After registering specs and plugins, check for mismatches
pm.check_pending()
# Raises PluginValidationError if there are issues
```

---

## Hook Implementations

Hook implementations are the actual functions provided by plugins. They must match the hookspec function signature (or a subset of it).

### Basic Implementation

```python
import pluggy

hookimpl = pluggy.HookimplMarker("myproject")

class SenderPlugin:
    """A plugin that processes items by sending them."""

    @hookimpl
    def process_item(self, item):
        send_to_endpoint(item)
        return {"sent": True, "item_id": item.id}
```

### Ordering: tryfirst and trylast

Pluggy calls implementations in **LIFO** (last-in, first-out) order by default -- the most recently registered plugin's implementation runs first. You can override this with `tryfirst` and `trylast`.

```python
class AuditPlugin:
    @hookimpl(trylast=True)
    def process_item(self, item):
        """Run after all other non-trylast implementations."""
        log_audit_trail(item)

class SecurityPlugin:
    @hookimpl(tryfirst=True)
    def process_item(self, item):
        """Run before all other non-tryfirst implementations."""
        if not is_authorized(item):
            raise PermissionError(f"Unauthorized: {item}")
```

**Execution order:**
1. `tryfirst=True` implementations (LIFO among themselves)
2. Normal implementations (LIFO)
3. `trylast=True` implementations (LIFO among themselves)

### wrapper=True -- Wrapping Other Implementations

Hook wrappers use a `yield`-based pattern to execute code before and after all other (non-wrapper) implementations. This replaced the older `hookimpl_tryfirst` pattern for cross-cutting concerns.

```python
class TimingPlugin:
    @hookimpl(wrapper=True)
    def process_item(self, item):
        """Time how long all other implementations take."""
        import time
        start = time.perf_counter()

        # yield suspends this wrapper and runs all inner implementations.
        # The result is the aggregated outcome from those implementations.
        result = yield

        elapsed = time.perf_counter() - start
        print(f"process_item took {elapsed:.3f}s")

        # You MUST return a value from a wrapper.
        # Return the original result, or a modified version.
        return result
```

### specname -- Implementing a Differently-Named Spec

Sometimes your plugin method name cannot match the hookspec name (e.g., name conflicts). Use `specname` to map it.

```python
class MyPlugin:
    @hookimpl(specname="process_item")
    def my_custom_process(self, item):
        """This implements the 'process_item' hook
        even though the method is named differently."""
        return transform(item)
```

### optionalhook=True -- Suppress Missing Spec Warning

If your plugin implements a hook that might not have a corresponding spec registered (e.g., optional integration), use `optionalhook=True` to silence the warning.

```python
class OptionalIntegrationPlugin:
    @hookimpl(optionalhook=True)
    def third_party_hook(self, data):
        """Implements a hook that may not exist in this setup."""
        return process(data)
```

---

## Hook Wrappers (In Depth)

Hook wrappers are one of pluggy's most powerful features. They allow a plugin to wrap the execution of all other implementations, providing **before/after/error** handling -- similar to middleware or context managers.

### The Yield-Based Pattern

```python
class WrapperPlugin:
    @hookimpl(wrapper=True)
    def my_hook(self, arg1, arg2):
        # ---- BEFORE phase ----
        # Code here runs before any non-wrapper implementations
        print("Before all implementations")

        # yield runs all inner implementations and captures the result
        # For normal hooks: result is a list of return values
        # For firstresult hooks: result is a single value
        result = yield

        # ---- AFTER phase ----
        # Code here runs after all non-wrapper implementations
        print("After all implementations")

        # You MUST return something from a wrapper
        return result
```

### Error Handling in Wrappers

Wrappers can catch exceptions from inner implementations:

```python
class ErrorHandlingPlugin:
    @hookimpl(wrapper=True)
    def process_item(self, item):
        try:
            result = yield
        except Exception as exc:
            log_error(f"process_item failed for {item}: {exc}")
            # Re-raise to propagate, or return a fallback
            raise
        else:
            return result
```

### Wrapper Ordering

Wrappers themselves can use `tryfirst` and `trylast`:

```python
class OuterWrapper:
    @hookimpl(wrapper=True, tryfirst=True)
    def process_item(self, item):
        """Outermost wrapper -- runs first, yields last."""
        print("outer before")
        result = yield
        print("outer after")
        return result

class InnerWrapper:
    @hookimpl(wrapper=True, trylast=True)
    def process_item(self, item):
        """Innermost wrapper -- runs last before real impls, yields first."""
        print("inner before")
        result = yield
        print("inner after")
        return result
```

**Execution flow:**
```
outer before  (OuterWrapper, tryfirst)
  inner before  (InnerWrapper, trylast)
    [actual implementations run here]
  inner after
outer after
```

### Real-World Wrapper: Transaction Management

```python
class TransactionPlugin:
    @hookimpl(wrapper=True)
    def process_item(self, item):
        """Wrap item processing in a database transaction."""
        db = get_database()
        transaction = db.begin()

        try:
            result = yield
        except Exception:
            transaction.rollback()
            raise
        else:
            transaction.commit()
            return result
```
