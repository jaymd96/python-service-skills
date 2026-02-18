# pluggy — Examples & Gotchas

> Part of the pluggy skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Hook Argument Mismatch Errors](#1-hook-argument-mismatch-errors)
  - [2. Ordering Subtleties with tryfirst/trylast](#2-ordering-subtleties-with-tryfirsttrylast)
  - [3. Result Collection: List vs. Single Value](#3-result-collection-list-vs-single-value)
  - [4. Plugin Registration Order Matters](#4-plugin-registration-order-matters)
  - [5. Historic Hooks and Late Registration](#5-historic-hooks-and-late-registration)
  - [6. Forgetting to Return from Wrappers](#6-forgetting-to-return-from-wrappers)
  - [7. Self Parameter in Specs vs. Implementations](#7-self-parameter-in-specs-vs-implementations)
  - [8. Registering the Same Plugin Twice](#8-registering-the-same-plugin-twice)
- [Complete Code Examples](#complete-code-examples)
  - [Example 1: Basic Hookspec + Hookimpl](#example-1-basic-hookspec--hookimpl)
  - [Example 2: Plugin Manager Setup and Usage](#example-2-plugin-manager-setup-and-usage)
  - [Example 3: Hook Wrappers for Cross-Cutting Concerns](#example-3-hook-wrappers-for-cross-cutting-concerns)
  - [Example 4: Entry Point Auto-Discovery](#example-4-entry-point-auto-discovery)
  - [Example 5: Real-World Plugin System Architecture](#example-5-real-world-plugin-system-architecture)
- [References](#references)

## Gotchas and Common Mistakes

### 1. Hook Argument Mismatch Errors

Pluggy is strict about argument names. All hook call arguments must match the hookspec signature. Extra or missing arguments cause errors.

```python
class Spec:
    @hookspec
    def my_hook(self, arg1, arg2):
        pass

# WRONG -- 'arg3' is not in the spec
pm.hook.my_hook(arg1="a", arg2="b", arg3="c")
# Raises: HookCallError

# WRONG -- calling with positional arguments
pm.hook.my_hook("a", "b")
# Raises: HookCallError (hooks must be called with keyword arguments)

# RIGHT
pm.hook.my_hook(arg1="a", arg2="b")
```

Implementations may accept a **subset** of the spec's arguments (pluggy will only pass the ones the implementation asks for):

```python
class Spec:
    @hookspec
    def my_hook(self, arg1, arg2, arg3):
        pass

class Plugin:
    @hookimpl
    def my_hook(self, arg1):
        # This is fine -- arg2 and arg3 are simply not passed
        return arg1
```

But an implementation **cannot** have arguments that the spec does not define:

```python
class BadPlugin:
    @hookimpl
    def my_hook(self, arg1, extra_arg):
        # PluginValidationError: extra_arg is not in the spec
        pass
```

### 2. Ordering Subtleties with tryfirst/trylast

Default ordering is **LIFO** (last registered = first called). This catches people off guard.

```python
pm.register(PluginA())  # Registered first
pm.register(PluginB())  # Registered second

# Call order: PluginB.hook() -> PluginA.hook()
# (LIFO, not FIFO!)

# tryfirst/trylast override registration order:
# 1. tryfirst implementations (LIFO among themselves)
# 2. normal implementations (LIFO among themselves)
# 3. trylast implementations (LIFO among themselves)
```

### 3. Result Collection: List vs. Single Value

For normal hooks, results are always a **list** -- even if only one plugin is registered:

```python
class Spec:
    @hookspec
    def get_value(self):
        pass

class OnlyPlugin:
    @hookimpl
    def get_value(self):
        return 42

pm.register(OnlyPlugin())
result = pm.hook.get_value()
# result == [42]  -- it's a list, NOT 42!
```

For `firstresult=True` hooks, the result is a **single value**:

```python
class Spec:
    @hookspec(firstresult=True)
    def get_value(self):
        pass

result = pm.hook.get_value()
# result == 42  -- single value, not a list
```

### 4. Plugin Registration Order Matters

Since pluggy uses LIFO ordering, the order you register plugins determines execution order. Be deliberate.

```python
# For a pipeline where order matters, register in reverse:
pm.register(last_step)
pm.register(middle_step)
pm.register(first_step)
# Execution: first_step -> middle_step -> last_step

# Or use tryfirst/trylast for explicit control (preferred):
class FirstStep:
    @hookimpl(tryfirst=True)
    def process(self, data):
        ...

class LastStep:
    @hookimpl(trylast=True)
    def process(self, data):
        ...
```

### 5. Historic Hooks and Late Registration

Historic hooks must be called with `call_historic()`, not via the normal `pm.hook.name()` path:

```python
class Spec:
    @hookspec(historic=True)
    def setup(self, config):
        pass

# WRONG -- this will NOT replay to late plugins
pm.hook.setup(config=my_config)

# RIGHT -- use call_historic for historic hooks
pm.hook.setup.call_historic(kwargs=dict(config=my_config))

# Now late-registered plugins will receive the call
pm.register(LatePlugin())  # LatePlugin.setup(config=my_config) is auto-called
```

Historic hooks do **not** collect return values. They are fire-and-forget by design. If you need results, provide a `result_callback`:

```python
results = []
pm.hook.setup.call_historic(
    kwargs=dict(config=my_config),
    result_callback=lambda res: results.append(res),
)
```

### 6. Forgetting to Return from Wrappers

A wrapper **must** return a value. If you forget, the hook result will be `None`:

```python
class BrokenWrapper:
    @hookimpl(wrapper=True)
    def my_hook(self, arg):
        result = yield
        # BUG: forgot to return result!
        # All callers will get None instead of the real result

class CorrectWrapper:
    @hookimpl(wrapper=True)
    def my_hook(self, arg):
        result = yield
        return result  # Always return!
```

### 7. Self Parameter in Specs vs. Implementations

The `self` parameter in hookspec class methods is **not** passed to hook calls. Hook calls only use the explicitly declared arguments:

```python
class Spec:
    @hookspec
    def my_hook(self, arg1, arg2):
        # 'self' is part of the class but NOT a hook argument
        pass

# Call with only the real arguments
pm.hook.my_hook(arg1="a", arg2="b")
# NOT: pm.hook.my_hook(self=x, arg1="a", arg2="b")
```

### 8. Registering the Same Plugin Twice

Registering the same plugin instance twice raises an error:

```python
plugin = MyPlugin()
pm.register(plugin)
pm.register(plugin)  # Raises: ValueError - plugin already registered
```

Use `pm.is_registered(plugin)` to check first if needed.

---

## Complete Code Examples

### Example 1: Basic Hookspec + Hookimpl

A minimal example showing the complete pluggy lifecycle.

```python
"""Minimal pluggy example: calculator with plugin-based operations."""
import pluggy

# Step 1: Define markers for our project
hookspec = pluggy.HookspecMarker("calculator")
hookimpl = pluggy.HookimplMarker("calculator")


# Step 2: Define the hook specification
class CalculatorSpec:
    """Hook specifications for the calculator."""

    @hookspec
    def compute(self, a, b):
        """Compute something with two numbers.

        Returns a dict with 'operation' and 'result' keys.
        """

    @hookspec(firstresult=True)
    def format_result(self, value):
        """Format a result value for display.

        First non-None result wins.
        """


# Step 3: Define plugin implementations
class AddPlugin:
    """Plugin that adds numbers."""

    @hookimpl
    def compute(self, a, b):
        return {"operation": "add", "result": a + b}


class MultiplyPlugin:
    """Plugin that multiplies numbers."""

    @hookimpl
    def compute(self, a, b):
        return {"operation": "multiply", "result": a * b}


class FancyFormatter:
    """Plugin that formats results nicely."""

    @hookimpl
    def format_result(self, value):
        return f">>> {value} <<<"


# Step 4: Wire everything together
def main():
    # Create the plugin manager
    pm = pluggy.PluginManager("calculator")

    # Register the spec
    pm.add_hookspecs(CalculatorSpec)

    # Register plugins
    pm.register(AddPlugin())
    pm.register(MultiplyPlugin())
    pm.register(FancyFormatter())

    # Call the hook -- all implementations run, results collected in a list
    results = pm.hook.compute(a=3, b=4)
    for r in results:
        print(f"{r['operation']}: {r['result']}")
    # Output:
    #   multiply: 12
    #   add: 7

    # Call firstresult hook -- stops at first non-None
    formatted = pm.hook.format_result(value=42)
    print(formatted)
    # Output: >>> 42 <<<


if __name__ == "__main__":
    main()
```

### Example 2: Plugin Manager Setup and Usage

A more structured example showing real-world plugin manager patterns.

```python
"""Structured plugin manager with registration, discovery, and lifecycle."""
import pluggy

hookspec = pluggy.HookspecMarker("dataprocessor")
hookimpl = pluggy.HookimplMarker("dataprocessor")


# --- Hook Specifications ---

class DataProcessorSpec:
    """All extension points for the data processor."""

    @hookspec
    def validate_record(self, record):
        """Validate a data record. Return list of error strings, or empty list."""

    @hookspec
    def transform_record(self, record):
        """Transform a record. Return the transformed record."""

    @hookspec(firstresult=True)
    def serialize_record(self, record, format):
        """Serialize a record to the given format. First handler wins."""

    @hookspec(historic=True)
    def processor_started(self, config):
        """Called when the processor starts. Historic: replays to late plugins."""


# --- Built-in Plugins ---

class SchemaValidator:
    """Validates records against a schema."""

    @hookimpl
    def validate_record(self, record):
        errors = []
        if "id" not in record:
            errors.append("missing required field: id")
        if "timestamp" not in record:
            errors.append("missing required field: timestamp")
        return errors


class NormalizationPlugin:
    """Normalizes record fields."""

    @hookimpl
    def transform_record(self, record):
        normalized = dict(record)
        if "name" in normalized:
            normalized["name"] = normalized["name"].strip().lower()
        return normalized


class JsonSerializer:
    """Serializes records to JSON."""

    @hookimpl
    def serialize_record(self, record, format):
        if format != "json":
            return None  # Pass to next implementation
        import json
        return json.dumps(record, default=str)


class CsvSerializer:
    """Serializes records to CSV."""

    @hookimpl
    def serialize_record(self, record, format):
        if format != "csv":
            return None  # Pass to next implementation
        return ",".join(str(v) for v in record.values())


# --- Plugin Manager Factory ---

def create_processor_manager(extra_plugins=None):
    """Create and configure the plugin manager.

    Args:
        extra_plugins: Optional list of additional plugin instances.

    Returns:
        Configured PluginManager.
    """
    pm = pluggy.PluginManager("dataprocessor")
    pm.add_hookspecs(DataProcessorSpec)

    # Register built-in plugins
    pm.register(SchemaValidator(), name="schema_validator")
    pm.register(NormalizationPlugin(), name="normalizer")
    pm.register(JsonSerializer(), name="json_serializer")
    pm.register(CsvSerializer(), name="csv_serializer")

    # Register any extra plugins
    if extra_plugins:
        for plugin in extra_plugins:
            pm.register(plugin)

    # Load third-party plugins from entry points
    pm.load_setuptools_entrypoints("dataprocessor")

    return pm


# --- Usage ---

def process_records(pm, records, output_format="json"):
    """Process a batch of records through the plugin pipeline."""

    # Notify plugins that processing has started
    pm.hook.processor_started.call_historic(
        kwargs=dict(config={"format": output_format, "count": len(records)})
    )

    output = []
    for record in records:
        # Validate
        all_errors = pm.hook.validate_record(record=record)
        # all_errors is a list of lists -- flatten
        errors = [e for error_list in all_errors for e in error_list]
        if errors:
            print(f"Skipping invalid record: {errors}")
            continue

        # Transform (each plugin returns a version; use the last one)
        transformed_versions = pm.hook.transform_record(record=record)
        final_record = transformed_versions[0] if transformed_versions else record

        # Serialize (firstresult -- first format match wins)
        serialized = pm.hook.serialize_record(
            record=final_record, format=output_format
        )
        if serialized:
            output.append(serialized)

    return output


if __name__ == "__main__":
    pm = create_processor_manager()

    records = [
        {"id": 1, "timestamp": "2024-01-01", "name": "  Alice  "},
        {"id": 2, "timestamp": "2024-01-02", "name": "  Bob  "},
        {"id": 3, "name": "No Timestamp"},  # Missing timestamp -- will fail validation
    ]

    results = process_records(pm, records, output_format="json")
    for r in results:
        print(r)
```

### Example 3: Hook Wrappers for Cross-Cutting Concerns

Demonstrates wrappers for timing, logging, error handling, and transactions.

```python
"""Hook wrappers for cross-cutting concerns: logging, timing, error handling."""
import pluggy
import time
import logging

hookspec = pluggy.HookspecMarker("webapp")
hookimpl = pluggy.HookimplMarker("webapp")

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("webapp")


# --- Specs ---

class WebAppSpec:
    @hookspec
    def handle_request(self, request):
        """Handle an incoming HTTP request. Return a response dict."""

    @hookspec(firstresult=True)
    def authenticate(self, token):
        """Authenticate a request token. Return user info or None."""


# --- Wrapper Plugins (cross-cutting concerns) ---

class TimingWrapper:
    """Measures execution time of all hook implementations."""

    @hookimpl(wrapper=True)
    def handle_request(self, request):
        start = time.perf_counter()
        result = yield
        elapsed = time.perf_counter() - start
        logger.info(f"handle_request took {elapsed:.4f}s for {request.get('path')}")
        return result


class LoggingWrapper:
    """Logs all request handling."""

    @hookimpl(wrapper=True, tryfirst=True)
    def handle_request(self, request):
        logger.info(f"Incoming request: {request.get('method')} {request.get('path')}")
        try:
            result = yield
        except Exception as exc:
            logger.error(f"Request failed: {exc}")
            raise
        else:
            logger.info(f"Request completed: {len(result)} response(s)")
            return result


class ErrorBoundaryWrapper:
    """Catches exceptions and returns error responses instead of crashing."""

    @hookimpl(wrapper=True)
    def handle_request(self, request):
        try:
            result = yield
        except PermissionError as exc:
            return [{"status": 403, "body": f"Forbidden: {exc}"}]
        except FileNotFoundError as exc:
            return [{"status": 404, "body": f"Not found: {exc}"}]
        except Exception as exc:
            return [{"status": 500, "body": f"Internal error: {exc}"}]
        else:
            return result


# --- Regular Plugins ---

class StaticFileHandler:
    """Serves static files."""

    @hookimpl
    def handle_request(self, request):
        if request.get("path", "").startswith("/static/"):
            filename = request["path"].replace("/static/", "")
            return {"status": 200, "body": f"[contents of {filename}]"}
        return None

class ApiHandler:
    """Handles API requests."""

    @hookimpl
    def handle_request(self, request):
        if request.get("path", "").startswith("/api/"):
            return {"status": 200, "body": '{"data": "api response"}'}
        return None

class TokenAuthenticator:
    """Simple token-based auth."""

    @hookimpl
    def authenticate(self, token):
        valid_tokens = {"abc123": {"user": "alice", "role": "admin"}}
        return valid_tokens.get(token)


# --- Application ---

def main():
    pm = pluggy.PluginManager("webapp")
    pm.add_hookspecs(WebAppSpec)

    # Register wrappers (order matters for nesting)
    pm.register(LoggingWrapper(), name="logging")
    pm.register(TimingWrapper(), name="timing")
    pm.register(ErrorBoundaryWrapper(), name="error_boundary")

    # Register handlers
    pm.register(StaticFileHandler(), name="static_files")
    pm.register(ApiHandler(), name="api")
    pm.register(TokenAuthenticator(), name="auth")

    # Simulate requests
    request = {"method": "GET", "path": "/api/users", "token": "abc123"}
    responses = pm.hook.handle_request(request=request)
    print(f"Responses: {responses}")

    # Authentication
    user = pm.hook.authenticate(token="abc123")
    print(f"Authenticated: {user}")

    user = pm.hook.authenticate(token="invalid")
    print(f"Authenticated: {user}")  # None


if __name__ == "__main__":
    main()
```

### Example 4: Entry Point Auto-Discovery

Demonstrates how to build a host application and a separate plugin package.

**Host application** (`myapp/core.py`):

```python
"""Host application that discovers plugins via entry points."""
import pluggy

hookspec = pluggy.HookspecMarker("myapp")
hookimpl = pluggy.HookimplMarker("myapp")


class MyAppSpec:
    """Extension points for myapp."""

    @hookspec
    def get_commands(self):
        """Return a list of CLI command dicts: [{"name": ..., "func": ...}]."""

    @hookspec(historic=True)
    def app_started(self, app):
        """Called when the app starts. Historic for late-loading plugins."""

    @hookspec
    def shutdown(self):
        """Called when the app is shutting down."""


class BuiltinPlugin:
    """Built-in commands that ship with the app."""

    @hookimpl
    def get_commands(self):
        return [
            {"name": "help", "func": lambda: print("Available commands: ...")},
            {"name": "version", "func": lambda: print("myapp v1.0.0")},
        ]


def create_app():
    pm = pluggy.PluginManager("myapp")
    pm.add_hookspecs(MyAppSpec)

    # Register built-in plugins first
    pm.register(BuiltinPlugin(), name="builtin")

    # Discover and load third-party plugins
    pm.load_setuptools_entrypoints("myapp")

    # Report loaded plugins
    print("Loaded plugins:")
    for plugin in pm.get_plugins():
        name = pm.get_name(plugin)
        print(f"  - {name}")

    for plugin, dist in pm.list_plugin_distinfo():
        print(f"  (from package: {dist.project_name}=={dist.version})")

    # Notify all plugins that the app has started
    pm.hook.app_started.call_historic(kwargs=dict(app={"name": "myapp"}))

    return pm


def run_command(pm, command_name):
    """Run a command by name, discovered from all plugins."""
    all_command_lists = pm.hook.get_commands()
    for command_list in all_command_lists:
        for cmd in command_list:
            if cmd["name"] == command_name:
                cmd["func"]()
                return
    print(f"Unknown command: {command_name}")
```

**Third-party plugin package** (`myapp-plugin-git/myapp_plugin_git/plugin.py`):

```python
"""Third-party git plugin for myapp, discovered via entry points."""
import pluggy

hookimpl = pluggy.HookimplMarker("myapp")


class GitPlugin:
    """Adds git-related commands to myapp."""

    @hookimpl
    def get_commands(self):
        return [
            {"name": "git-status", "func": self._status},
            {"name": "git-log", "func": self._log},
        ]

    @hookimpl
    def app_started(self, app):
        print(f"GitPlugin initialized for {app['name']}")

    @hookimpl
    def shutdown(self):
        print("GitPlugin shutting down")

    def _status(self):
        print("On branch main, nothing to commit")

    def _log(self):
        print("abc1234 Initial commit")
```

**Plugin package configuration** (`myapp-plugin-git/pyproject.toml`):

```toml
[project]
name = "myapp-plugin-git"
version = "0.1.0"
dependencies = ["pluggy>=1.5.0"]

[project.entry-points."myapp"]
git = "myapp_plugin_git.plugin:GitPlugin"
```

### Example 5: Real-World Plugin System Architecture

A complete, production-quality plugin system with lifecycle management, configuration, and error isolation.

```python
"""
Real-world plugin architecture: a document processing pipeline.

Demonstrates:
- Structured plugin manager with lifecycle
- Historic hooks for configuration
- Hook wrappers for error isolation and metrics
- firstresult for format negotiation
- Plugin introspection and health checks
"""
import pluggy
import logging
import time
from dataclasses import dataclass, field
from typing import Any

hookspec = pluggy.HookspecMarker("docpipeline")
hookimpl = pluggy.HookimplMarker("docpipeline")

logger = logging.getLogger("docpipeline")


# =============================================================================
# Data Types
# =============================================================================

@dataclass
class Document:
    id: str
    content: str
    metadata: dict = field(default_factory=dict)
    annotations: list = field(default_factory=list)


@dataclass
class PipelineResult:
    document: Document
    stages_applied: list = field(default_factory=list)
    errors: list = field(default_factory=list)
    duration_seconds: float = 0.0


# =============================================================================
# Hook Specifications
# =============================================================================

class DocPipelineSpec:
    """All extension points for the document pipeline."""

    @hookspec(historic=True)
    def pipeline_configure(self, config):
        """Configure the plugin with pipeline settings.

        Historic: plugins registered after startup still receive config.
        """

    @hookspec
    def validate_document(self, document):
        """Validate a document before processing.

        Returns: list of error strings (empty = valid).
        """

    @hookspec
    def process_document(self, document):
        """Process/transform a document.

        Modify the document in-place or return annotations.
        Returns: dict of annotations to add, or None.
        """

    @hookspec(firstresult=True)
    def detect_language(self, text):
        """Detect the language of text. First confident answer wins.

        Returns: language code string (e.g., 'en') or None to defer.
        """

    @hookspec
    def on_document_complete(self, document, result):
        """Called after a document has been fully processed."""

    @hookspec
    def get_plugin_info(self):
        """Return plugin metadata for introspection.

        Returns: dict with 'name', 'version', 'description' keys.
        """


# =============================================================================
# Core Wrappers (cross-cutting concerns)
# =============================================================================

class MetricsWrapper:
    """Collects processing metrics."""

    def __init__(self):
        self.total_processed = 0
        self.total_errors = 0
        self.total_time = 0.0

    @hookimpl(wrapper=True)
    def process_document(self, document):
        start = time.perf_counter()
        try:
            result = yield
        except Exception:
            self.total_errors += 1
            raise
        else:
            self.total_processed += 1
            self.total_time += time.perf_counter() - start
            return result

    @hookimpl
    def get_plugin_info(self):
        return {
            "name": "metrics",
            "version": "1.0.0",
            "description": "Collects processing metrics",
            "stats": {
                "processed": self.total_processed,
                "errors": self.total_errors,
                "total_time": self.total_time,
            },
        }


class ErrorIsolationWrapper:
    """Prevents one plugin's error from crashing the entire pipeline."""

    @hookimpl(wrapper=True, tryfirst=True)
    def process_document(self, document):
        try:
            result = yield
        except Exception as exc:
            logger.error(
                f"Pipeline error processing document {document.id}: {exc}",
                exc_info=True,
            )
            # Return empty result instead of crashing
            return [None]
        else:
            return result


# =============================================================================
# Processing Plugins
# =============================================================================

class LengthValidator:
    """Validates document length."""

    def __init__(self, max_length=100_000):
        self.max_length = max_length

    @hookimpl
    def pipeline_configure(self, config):
        self.max_length = config.get("max_document_length", self.max_length)

    @hookimpl
    def validate_document(self, document):
        errors = []
        if not document.content:
            errors.append("Document content is empty")
        elif len(document.content) > self.max_length:
            errors.append(
                f"Document exceeds max length: "
                f"{len(document.content)} > {self.max_length}"
            )
        return errors

    @hookimpl
    def get_plugin_info(self):
        return {
            "name": "length_validator",
            "version": "1.0.0",
            "description": f"Validates document length (max {self.max_length})",
        }


class WordCounter:
    """Counts words and adds word_count annotation."""

    @hookimpl
    def process_document(self, document):
        words = document.content.split()
        return {"word_count": len(words)}

    @hookimpl
    def get_plugin_info(self):
        return {
            "name": "word_counter",
            "version": "1.0.0",
            "description": "Counts words in documents",
        }


class SimpleLanguageDetector:
    """Basic language detection based on common words."""

    INDICATORS = {
        "en": {"the", "is", "and", "of", "to", "in", "it", "that"},
        "es": {"el", "la", "de", "en", "que", "los", "del", "las"},
        "fr": {"le", "la", "de", "et", "les", "des", "en", "du"},
    }

    @hookimpl
    def detect_language(self, text):
        words = set(text.lower().split())
        scores = {}
        for lang, indicators in self.INDICATORS.items():
            scores[lang] = len(words & indicators)
        if scores:
            best = max(scores, key=scores.get)
            if scores[best] > 2:  # Confidence threshold
                return best
        return None  # Not confident, defer to next detector

    @hookimpl
    def get_plugin_info(self):
        return {
            "name": "simple_language_detector",
            "version": "1.0.0",
            "description": "Basic language detection via word frequency",
        }


class KeywordExtractor:
    """Extracts keywords from documents."""

    def __init__(self):
        self.stop_words = set()

    @hookimpl
    def pipeline_configure(self, config):
        self.stop_words = set(config.get("stop_words", []))

    @hookimpl(trylast=True)
    def process_document(self, document):
        words = document.content.lower().split()
        # Simple keyword extraction: most frequent non-stop words
        freq = {}
        for word in words:
            word = word.strip(".,!?;:")
            if word and word not in self.stop_words and len(word) > 3:
                freq[word] = freq.get(word, 0) + 1
        top_keywords = sorted(freq, key=freq.get, reverse=True)[:10]
        return {"keywords": top_keywords}

    @hookimpl
    def get_plugin_info(self):
        return {
            "name": "keyword_extractor",
            "version": "1.0.0",
            "description": "Extracts top keywords from documents",
        }


# =============================================================================
# Pipeline Manager
# =============================================================================

class DocumentPipeline:
    """Manages the document processing pipeline."""

    def __init__(self, config: dict[str, Any] | None = None):
        self.config = config or {}
        self.pm = pluggy.PluginManager("docpipeline")
        self.pm.add_hookspecs(DocPipelineSpec)

        # Register core wrappers
        self._metrics = MetricsWrapper()
        self.pm.register(ErrorIsolationWrapper(), name="error_isolation")
        self.pm.register(self._metrics, name="metrics")

        # Register built-in plugins
        self.pm.register(LengthValidator(), name="length_validator")
        self.pm.register(WordCounter(), name="word_counter")
        self.pm.register(SimpleLanguageDetector(), name="language_detector")
        self.pm.register(KeywordExtractor(), name="keyword_extractor")

        # Load third-party plugins
        self.pm.load_setuptools_entrypoints("docpipeline")

        # Send configuration (historic -- late plugins still get it)
        self.pm.hook.pipeline_configure.call_historic(
            kwargs=dict(config=self.config)
        )

    def process(self, document: Document) -> PipelineResult:
        """Run a document through the entire processing pipeline."""
        start = time.perf_counter()
        result = PipelineResult(document=document)

        # Step 1: Validate
        all_errors = self.pm.hook.validate_document(document=document)
        errors = [e for error_list in all_errors for e in error_list]
        if errors:
            result.errors = errors
            return result

        # Step 2: Detect language
        lang = self.pm.hook.detect_language(text=document.content)
        if lang:
            document.metadata["language"] = lang
            result.stages_applied.append("language_detection")

        # Step 3: Process (all plugins contribute annotations)
        annotation_results = self.pm.hook.process_document(document=document)
        for annotations in annotation_results:
            if annotations:
                document.annotations.append(annotations)
                result.stages_applied.append(
                    annotations.get("_stage", "processing")
                )

        # Step 4: Notify completion
        result.duration_seconds = time.perf_counter() - start
        self.pm.hook.on_document_complete(document=document, result=result)

        return result

    def get_plugin_info(self) -> list[dict]:
        """Get information about all registered plugins."""
        return self.pm.hook.get_plugin_info()

    @property
    def metrics(self) -> dict:
        """Current pipeline metrics."""
        info = [i for i in self.get_plugin_info() if i["name"] == "metrics"]
        return info[0]["stats"] if info else {}


# =============================================================================
# Usage
# =============================================================================

if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)

    # Create the pipeline with configuration
    pipeline = DocumentPipeline(config={
        "max_document_length": 50_000,
        "stop_words": ["the", "is", "and", "of", "to", "in", "it", "a", "an"],
    })

    # Show loaded plugins
    print("Loaded plugins:")
    for info in pipeline.get_plugin_info():
        print(f"  {info['name']} v{info['version']}: {info['description']}")

    # Process a document
    doc = Document(
        id="doc-001",
        content=(
            "The quick brown fox jumps over the lazy dog. "
            "This is a simple document that demonstrates the pipeline. "
            "The pipeline processes documents through multiple stages."
        ),
    )

    result = pipeline.process(doc)

    print(f"\nDocument: {doc.id}")
    print(f"  Language: {doc.metadata.get('language', 'unknown')}")
    print(f"  Annotations: {doc.annotations}")
    print(f"  Stages: {result.stages_applied}")
    print(f"  Duration: {result.duration_seconds:.4f}s")
    print(f"  Errors: {result.errors}")

    print(f"\nPipeline metrics: {pipeline.metrics}")
```

---

## References

- **Documentation:** https://pluggy.readthedocs.io/en/stable/
- **GitHub:** https://github.com/pytest-dev/pluggy
- **PyPI:** https://pypi.org/project/pluggy/
- **Changelog:** https://github.com/pytest-dev/pluggy/blob/main/CHANGELOG.rst
- **pytest plugin guide:** https://docs.pytest.org/en/stable/how-to/writing_plugins.html
