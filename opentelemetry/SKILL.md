---
name: opentelemetry
description: Vendor-neutral distributed tracing, metrics, and observability. Use when instrumenting services with traces/spans, collecting metrics, propagating context across microservices, or exporting telemetry to backends. Triggers on opentelemetry, tracing, spans, metrics, distributed tracing, OTLP, observability, instrumentation.
---

# OpenTelemetry — Distributed Tracing & Metrics (v1.29.0)

## Quick Start

```bash
pip install opentelemetry-api opentelemetry-sdk
pip install opentelemetry-exporter-otlp  # production exporter
```

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.resources import Resource

provider = TracerProvider(resource=Resource.create({"service.name": "my-service"}))
trace.set_tracer_provider(provider)
tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("my-operation") as span:
    span.set_attribute("user.id", 42)
    # ... your code here
```

## Key Patterns

### Creating spans with attributes
```python
tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("operation") as span:
    span.set_attribute("key", "value")
    span.add_event("checkpoint", {"detail": "info"})

# Nested spans auto-link parent-child
with tracer.start_as_current_span("parent"):
    with tracer.start_as_current_span("child"):
        pass  # child's parent is "parent"
```

### Production export with BatchSpanProcessor
```python
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace.export import BatchSpanProcessor

provider.add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="http://collector:4317"))
)
```

## References

- **[api.md](references/api.md)** — Tracing, metrics, span processors, exporters, context propagation, baggage, resources, and auto-instrumentation
- **[examples.md](references/examples.md)** — Full Flask tracing setup, custom metrics, distributed tracing across services, and common mistakes
