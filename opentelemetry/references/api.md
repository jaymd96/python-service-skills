# OpenTelemetry — API Reference

> Part of the opentelemetry skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Core API -- Tracing](#core-api----tracing)
  - [TracerProvider and Tracer](#tracerprovider-and-tracer)
  - [Creating Spans](#creating-spans)
  - [Span Attributes](#span-attributes)
  - [Span Events](#span-events)
  - [Span Status](#span-status)
  - [Recording Exceptions](#recording-exceptions)
  - [Span Kind](#span-kind)
- [Core API -- Metrics](#core-api----metrics)
  - [MeterProvider and Meter](#meterprovider-and-meter)
  - [Instrument Types](#instrument-types)
- [Span Processors](#span-processors)
  - [BatchSpanProcessor](#batchspanprocessor)
  - [SimpleSpanProcessor](#simplespanprocessor)
- [Exporters](#exporters)
  - [Console Exporter (Development)](#console-exporter-development)
  - [OTLP Exporter (Production)](#otlp-exporter-production)
  - [OTLP Metrics Exporter](#otlp-metrics-exporter)
- [Context Propagation](#context-propagation)
  - [W3C Trace Context (Default)](#w3c-trace-context-default)
  - [Injecting Context into Outgoing Requests](#injecting-context-into-outgoing-requests)
  - [Extracting Context from Incoming Requests](#extracting-context-from-incoming-requests)
- [Baggage](#baggage)
- [Resource](#resource)
- [Auto-Instrumentation](#auto-instrumentation)

## Core API -- Tracing

### TracerProvider and Tracer

The `TracerProvider` is the entry point for creating tracers. A `Tracer` creates spans.

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.resources import Resource

# Create a resource describing this service
resource = Resource.create({
    "service.name": "my-service",
    "service.version": "1.0.0",
    "deployment.environment": "production",
})

# Set up the tracer provider
provider = TracerProvider(resource=resource)
trace.set_tracer_provider(provider)

# Get a tracer
tracer = trace.get_tracer("my.module.name", "1.0.0")
```

### Creating Spans

#### `tracer.start_as_current_span()`

The most common way to create spans. Sets the span as the current span in context and automatically ends it.

```python
tracer = trace.get_tracer(__name__)

def process_order(order_id: str):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)
        validate_order(order_id)
        charge_payment(order_id)
        ship_order(order_id)

def validate_order(order_id: str):
    with tracer.start_as_current_span("validate_order") as span:
        span.set_attribute("order.id", order_id)
        # This span is automatically a child of "process_order"
        ...
```

#### Span Attributes

Attributes are key-value pairs attached to a span. Keys are strings; values can be strings, booleans, integers, floats, or sequences thereof.

```python
with tracer.start_as_current_span("http_request") as span:
    span.set_attribute("http.method", "GET")
    span.set_attribute("http.url", "https://api.example.com/data")
    span.set_attribute("http.status_code", 200)
    span.set_attribute("custom.retry_count", 3)

    # Set multiple attributes at once
    span.set_attributes({
        "db.system": "postgresql",
        "db.statement": "SELECT * FROM users",
        "db.operation": "SELECT",
    })
```

#### Span Events

Events are time-stamped annotations within a span.

```python
with tracer.start_as_current_span("process_batch") as span:
    span.add_event("batch_started", {"batch.size": 100})

    for i, item in enumerate(batch):
        process(item)

    span.add_event("batch_completed", {
        "batch.processed": 100,
        "batch.failed": 2,
    })
```

#### Span Status

Set the span status to indicate success or error.

```python
from opentelemetry.trace import StatusCode

with tracer.start_as_current_span("risky_operation") as span:
    try:
        result = do_work()
        span.set_status(StatusCode.OK)
    except Exception as e:
        span.set_status(StatusCode.ERROR, str(e))
        span.record_exception(e)
        raise
```

#### Recording Exceptions

```python
with tracer.start_as_current_span("operation") as span:
    try:
        risky_call()
    except Exception as e:
        span.record_exception(e)  # Adds exception as a span event
        span.set_status(StatusCode.ERROR, str(e))
        raise
```

#### Span Kind

```python
from opentelemetry.trace import SpanKind

# Server span (handles incoming requests)
with tracer.start_as_current_span("handle_request", kind=SpanKind.SERVER):
    ...

# Client span (makes outgoing requests)
with tracer.start_as_current_span("call_service", kind=SpanKind.CLIENT):
    ...

# Producer span (sends messages)
with tracer.start_as_current_span("send_message", kind=SpanKind.PRODUCER):
    ...

# Consumer span (receives messages)
with tracer.start_as_current_span("process_message", kind=SpanKind.CONSUMER):
    ...

# Internal span (default, internal operations)
with tracer.start_as_current_span("compute", kind=SpanKind.INTERNAL):
    ...
```

---

## Core API -- Metrics

### MeterProvider and Meter

```python
from opentelemetry import metrics
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.resources import Resource

resource = Resource.create({"service.name": "my-service"})
provider = MeterProvider(resource=resource)
metrics.set_meter_provider(provider)

meter = metrics.get_meter("my.module.name", "1.0.0")
```

### Instrument Types

#### Counter

A monotonically increasing value. Use for counts of events, requests, errors, etc.

```python
request_counter = meter.create_counter(
    name="http.requests",
    description="Total HTTP requests",
    unit="1",
)

request_counter.add(1, {"http.method": "GET", "http.route": "/api/users"})
```

#### UpDownCounter

A counter that can increase or decrease. Use for active connections, queue depth, etc.

```python
active_connections = meter.create_up_down_counter(
    name="db.connections.active",
    description="Active database connections",
)

active_connections.add(1)   # Connection opened
active_connections.add(-1)  # Connection closed
```

#### Histogram

Records a distribution of values. Use for request latencies, payload sizes, etc.

```python
latency_histogram = meter.create_histogram(
    name="http.request.duration",
    description="HTTP request latency",
    unit="ms",
)

import time
start = time.monotonic()
handle_request()
duration = (time.monotonic() - start) * 1000
latency_histogram.record(duration, {"http.method": "GET", "http.route": "/api"})
```

#### Gauge (Observable)

Reports the current value at collection time via a callback.

```python
import psutil

def cpu_usage_callback(options):
    yield metrics.Observation(psutil.cpu_percent(), {"cpu": "total"})

meter.create_observable_gauge(
    name="system.cpu.usage",
    description="CPU usage percentage",
    unit="%",
    callbacks=[cpu_usage_callback],
)
```

#### Observable Counter

An asynchronous counter that reports cumulative values via callbacks.

```python
def total_requests_callback(options):
    yield metrics.Observation(get_total_request_count(), {})

meter.create_observable_counter(
    name="http.requests.total",
    description="Total requests from external counter",
    callbacks=[total_requests_callback],
)
```

---

## Span Processors

Span processors handle spans after they are created and before they are exported.

### `BatchSpanProcessor`

Batches spans and exports them periodically. This is the recommended processor for production.

```python
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

exporter = OTLPSpanExporter(endpoint="http://localhost:4317")
processor = BatchSpanProcessor(
    exporter,
    max_queue_size=2048,
    schedule_delay_millis=5000,
    max_export_batch_size=512,
    export_timeout_millis=30000,
)
provider.add_span_processor(processor)
```

### `SimpleSpanProcessor`

Exports each span immediately, synchronously. Only use for development or testing.

```python
from opentelemetry.sdk.trace.export import SimpleSpanProcessor, ConsoleSpanExporter

processor = SimpleSpanProcessor(ConsoleSpanExporter())
provider.add_span_processor(processor)
```

---

## Exporters

### Console Exporter (Development)

```python
from opentelemetry.sdk.trace.export import ConsoleSpanExporter, SimpleSpanProcessor

provider.add_span_processor(SimpleSpanProcessor(ConsoleSpanExporter()))
```

### OTLP Exporter (Production)

The standard exporter for sending data to any OTLP-compatible backend (Jaeger, Grafana Tempo, Datadog, etc.).

```python
# gRPC (default, recommended)
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
exporter = OTLPSpanExporter(endpoint="http://localhost:4317")

# HTTP/protobuf
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
exporter = OTLPSpanExporter(endpoint="http://localhost:4318/v1/traces")
```

### OTLP Metrics Exporter

```python
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader

metric_reader = PeriodicExportingMetricReader(
    OTLPMetricExporter(endpoint="http://localhost:4317"),
    export_interval_millis=60000,
)

provider = MeterProvider(resource=resource, metric_readers=[metric_reader])
```

---

## Context Propagation

Context propagation carries trace context (trace ID, span ID) across service boundaries.

### W3C Trace Context (Default)

```python
from opentelemetry.propagate import set_global_textmap
from opentelemetry.propagators.composite import CompositePropagator
from opentelemetry.trace.propagation import TraceContextTextMapPropagator
from opentelemetry.baggage.propagation import W3CBaggagePropagator

set_global_textmap(CompositePropagator([
    TraceContextTextMapPropagator(),
    W3CBaggagePropagator(),
]))
```

### Injecting Context into Outgoing Requests

```python
from opentelemetry.propagate import inject

headers = {}
inject(headers)
# headers now contains: {"traceparent": "00-<trace_id>-<span_id>-01", ...}

import requests
response = requests.get("http://downstream-service/api", headers=headers)
```

### Extracting Context from Incoming Requests

```python
from opentelemetry.propagate import extract

# In a web framework handler:
context = extract(request.headers)
with tracer.start_as_current_span("handle", context=context):
    ...
```

---

## Baggage

Baggage carries user-defined key-value pairs across service boundaries alongside trace context.

```python
from opentelemetry import baggage, context

# Set baggage
ctx = baggage.set_baggage("user.id", "12345")
context.attach(ctx)

# Read baggage (in this or a downstream service)
user_id = baggage.get_baggage("user.id")

# Get all baggage
all_baggage = baggage.get_all()
```

---

## Resource

A `Resource` describes the entity producing telemetry (e.g., your service).

```python
from opentelemetry.sdk.resources import Resource, SERVICE_NAME

resource = Resource.create({
    SERVICE_NAME: "checkout-service",
    "service.version": "2.5.1",
    "deployment.environment": "production",
    "host.name": "server-01",
    "service.instance.id": "checkout-01",
})
```

---

## Auto-Instrumentation

Auto-instrumentation automatically instruments popular libraries without code changes.

```bash
# Install the distro and bootstrap instrumentations
pip install opentelemetry-distro
opentelemetry-bootstrap -a install

# Run your application with auto-instrumentation
opentelemetry-instrument \
    --service_name my-service \
    --traces_exporter otlp \
    --metrics_exporter otlp \
    --exporter_otlp_endpoint http://localhost:4317 \
    python app.py
```

### Programmatic Auto-Instrumentation

```python
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

FlaskInstrumentor().instrument()
RequestsInstrumentor().instrument()
SQLAlchemyInstrumentor().instrument()
```
