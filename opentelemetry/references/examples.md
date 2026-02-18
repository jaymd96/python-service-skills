# OpenTelemetry — Examples & Gotchas

> Part of the opentelemetry skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Complete Code Examples](#complete-code-examples)
  - [Full Tracing Setup for a Flask Application](#full-tracing-setup-for-a-flask-application)
  - [Custom Metrics with Histograms and Counters](#custom-metrics-with-histograms-and-counters)
  - [Distributed Tracing Across Services](#distributed-tracing-across-services)
- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Forgetting to Set the Global Provider](#1-forgetting-to-set-the-global-provider)
  - [2. Using SimpleSpanProcessor in Production](#2-using-simplespanprocessor-in-production)
  - [3. Not Calling provider.shutdown()](#3-not-calling-providershutdown)
  - [4. Attribute Value Types](#4-attribute-value-types)
  - [5. API vs SDK Confusion](#5-api-vs-sdk-confusion)
  - [6. Span Names Should Be Low Cardinality](#6-span-names-should-be-low-cardinality)
  - [7. OTLP Exporter Port Confusion](#7-otlp-exporter-port-confusion)
  - [8. Missing record_exception Before Setting Error Status](#8-missing-record_exception-before-setting-error-status)
  - [9. Context Not Propagating in Thread Pools](#9-context-not-propagating-in-thread-pools)

## Complete Code Examples

### Full Tracing Setup for a Flask Application

```python
from flask import Flask
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.resources import Resource, SERVICE_NAME
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

# Configure resource and provider
resource = Resource.create({SERVICE_NAME: "my-flask-app"})
provider = TracerProvider(resource=resource)
provider.add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="http://localhost:4317"))
)
trace.set_tracer_provider(provider)

# Auto-instrument libraries
FlaskInstrumentor().instrument()
RequestsInstrumentor().instrument()

app = Flask(__name__)
tracer = trace.get_tracer(__name__)

@app.route("/orders/<order_id>")
def get_order(order_id):
    with tracer.start_as_current_span("fetch_order") as span:
        span.set_attribute("order.id", order_id)
        order = db_get_order(order_id)
        if order is None:
            span.set_status(trace.StatusCode.ERROR, "Order not found")
            return {"error": "not found"}, 404
        return order
```

### Custom Metrics with Histograms and Counters

```python
from opentelemetry import metrics
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import (
    PeriodicExportingMetricReader,
    ConsoleMetricExporter,
)
from opentelemetry.sdk.resources import Resource
import time

resource = Resource.create({"service.name": "metrics-demo"})
reader = PeriodicExportingMetricReader(ConsoleMetricExporter(), export_interval_millis=5000)
provider = MeterProvider(resource=resource, metric_readers=[reader])
metrics.set_meter_provider(provider)

meter = metrics.get_meter(__name__)

request_count = meter.create_counter("app.request.count", description="Total requests")
request_duration = meter.create_histogram("app.request.duration_ms", description="Request latency in ms")
active_users = meter.create_up_down_counter("app.users.active", description="Active users")

def handle_request(method, route):
    start = time.monotonic()
    request_count.add(1, {"method": method, "route": route})
    active_users.add(1)
    try:
        process_request()
    finally:
        duration = (time.monotonic() - start) * 1000
        request_duration.record(duration, {"method": method, "route": route})
        active_users.add(-1)
```

### Distributed Tracing Across Services

```python
# --- Service A (caller) ---
import requests
from opentelemetry import trace
from opentelemetry.propagate import inject

tracer = trace.get_tracer("service-a")

def call_service_b():
    with tracer.start_as_current_span("call_service_b", kind=trace.SpanKind.CLIENT) as span:
        headers = {}
        inject(headers)  # Injects traceparent header
        span.set_attribute("peer.service", "service-b")
        response = requests.get("http://service-b:8080/api/data", headers=headers)
        span.set_attribute("http.status_code", response.status_code)
        return response.json()

# --- Service B (receiver) ---
from opentelemetry.propagate import extract

def handle_request(request):
    context = extract(request.headers)
    with tracer.start_as_current_span("handle_request", context=context, kind=trace.SpanKind.SERVER):
        return process_data()
```

---

## Gotchas and Common Mistakes

### 1. Forgetting to Set the Global Provider

If you create a `TracerProvider` but never call `trace.set_tracer_provider(provider)`, all tracing calls become no-ops using the default no-op provider.

### 2. Using `SimpleSpanProcessor` in Production

`SimpleSpanProcessor` exports spans synchronously, blocking your application code. Always use `BatchSpanProcessor` in production.

### 3. Not Calling `provider.shutdown()`

Failing to call `provider.shutdown()` on application exit can cause spans in the batch queue to be lost.

```python
import atexit
atexit.register(provider.shutdown)
```

### 4. Attribute Value Types

Span attributes only accept strings, booleans, integers, floats, and sequences of these types. Passing dicts, None, or custom objects will be silently dropped.

```python
# WRONG: dict values are ignored
span.set_attribute("data", {"key": "value"})

# CORRECT: use flat keys
span.set_attribute("data.key", "value")
```

### 5. API vs SDK Confusion

`opentelemetry-api` is the interface; `opentelemetry-sdk` is the implementation. If you only install the API, everything works but produces no telemetry (no-op). You need both packages.

### 6. Span Names Should Be Low Cardinality

Span names should describe the operation type, not include variable data like IDs.

```python
# WRONG: high cardinality span name
tracer.start_as_current_span(f"GET /users/{user_id}")

# CORRECT: low cardinality with attributes
with tracer.start_as_current_span("GET /users/{id}") as span:
    span.set_attribute("user.id", user_id)
```

### 7. OTLP Exporter Port Confusion

gRPC uses port 4317; HTTP uses port 4318. Mixing them up causes silent failures.

### 8. Missing `record_exception` Before Setting Error Status

Calling `set_status(StatusCode.ERROR)` does not automatically record the exception. Call `record_exception()` explicitly to capture the stack trace.

### 9. Context Not Propagating in Thread Pools

OpenTelemetry context does not automatically propagate to threads in `ThreadPoolExecutor`. Use `context.attach()` manually or the OTel context-aware executor wrapper.

```python
from opentelemetry import context

current_context = context.get_current()

def task():
    token = context.attach(current_context)
    try:
        with tracer.start_as_current_span("task"):
            ...
    finally:
        context.detach(token)
```
