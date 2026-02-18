# Enchant — Type-Safe Code Generation for Apollo-on-k3s

## Overview

**Enchant** is a "define once, generate everywhere" code generation framework. Developers author API contracts in a constrained YAML DSL (`.enchant.yml`), and Enchant compiles them through an internal OpenAPI 3.1 representation into production-ready Python code: **attrs models**, **abstract server stubs**, and **typed httpx clients** with built-in resilience.

| Property | Value |
|----------|-------|
| **Products** | `enchant-spec`, `enchant-codegen`, `enchant-dialogue`, `enchant-verify`, `enchant-cli`, `pants-enchant` |
| **Version** | 0.1.0 |
| **License** | MIT |
| **Python** | >=3.11 |
| **Repository** | witchcraft/enchant |

### Why Enchant for Apollo-on-k3s

| Capability | Apollo Use Case |
|------------|-----------------|
| Single source of truth | One `.enchant.yml` defines all API contracts |
| Generated attrs models | Type-safe domain objects with cattrs serialization |
| Generated server stubs | Abstract base classes for API implementation |
| Generated clients | Typed httpx clients with AIMD, retry, tracing |
| Safety annotations | Safe/unsafe/do-not-log on every field |
| Conformance testing | Proves generated code matches wire spec |
| Forward compatibility | Clients tolerate unknown enum values, unknown union variants |
| Constraint validation | min-length, max-length, pattern, numeric bounds from DSL |

### Architecture

```
.enchant.yml (developer writes YAML)
         │
         ▼
   enchant-spec (YAML → OpenAPI 3.1 in memory)
         │
    ┌────┼────────────────┐
    ▼    ▼                ▼
gen-models  gen-server  gen-client
    │        │              │
    ▼        ▼              ▼
 attrs    abstract      httpx client
classes   base +         wrapping
+ cattrs  router fn     EnchantChannel
```

---

## Product Breakdown

### 1. enchant-spec (DSL Compiler)

Compiles `.enchant.yml` YAML into an OpenAPI 3.1 dict. The OpenAPI is an internal representation — developers never author or see it.

**Dependencies:** `pyyaml`

**Public API:**

```python
from enchant_spec import compile_to_openapi, compile_to_ast

# Compile YAML files to OpenAPI dict
openapi = compile_to_openapi(["service.enchant.yml", "shared-types.enchant.yml"])

# Compile to intermediate AST (for tooling)
definition = compile_to_ast(["service.enchant.yml"])
```

### 2. enchant-codegen (Code Generators)

Three independent generators, all consuming OpenAPI and producing Python.

**Dependencies:** `jinja2`

#### gen-models — attrs Classes + cattrs Converter

Generates frozen attrs classes with validators, aliases, and safety metadata.

```python
# Generated output example
import attrs
from typing import Annotated
from enchant_runtime import SafeArg, UnsafeArg

@attrs.define(frozen=True, slots=True)
class SearchRequest:
    origin: Annotated[str, SafeArg] = attrs.field(
        metadata={"alias": "origin", "safety": "safe"}
    )
    destination: Annotated[str, SafeArg] = attrs.field(
        metadata={"alias": "destination", "safety": "safe"}
    )
    passenger_name: Annotated[str, UnsafeArg] = attrs.field(
        metadata={"alias": "passengerName", "safety": "unsafe"}
    )
```

#### gen-server — Abstract Base + Router

Generates abstract base classes that developers implement, plus a registration function for wiring into the router.

```python
# Generated output example
from abc import ABC, abstractmethod

class FlightSearchServiceBase(ABC):
    @abstractmethod
    async def search_flights(
        self,
        origin: str,
        destination: str,
        departure_date: str,
    ) -> list[FlightResult]:
        ...

    @abstractmethod
    async def get_flight(self, flight_id: str) -> FlightResult:
        ...

def register_flight_search_service(router, impl: FlightSearchServiceBase):
    """Wire implementation into HTTP router."""
    ...
```

#### gen-client — Typed httpx Client

Generates thin client wrappers that delegate to `EnchantChannel` for resilience.

```python
# Generated output example
class FlightSearchServiceClient:
    def __init__(self, channel: EnchantChannel):
        self._channel = channel

    def search_flights(
        self, origin: str, destination: str, departure_date: str
    ) -> list[FlightResult]:
        return self._channel.execute(
            method="GET",
            path=f"/api/flights/search",
            params={"origin": origin, "destination": destination, "departureDate": departure_date},
            response_type=list[FlightResult],
        )
```

### 3. enchant-dialogue (Client Runtime)

The runtime library that generated clients depend on. Provides resilience, observability, and safety.

**Dependencies:** `httpx`, `cattrs`, `structlog`, `opentelemetry` (optional)

**Public API:**

```python
from enchant_dialogue import (
    EnchantChannel,          # Main HTTP channel with resilience
    ChannelConfig,           # Configuration
    EnchantRemoteError,      # Server returned error
    EnchantTimeoutError,     # Request timed out
    EnchantTransportError,   # Connection failed
    SafeArg,                 # Marker: safe for logging
    UnsafeArg,               # Marker: redact in logs
)
```

#### Features

**AIMD Concurrency Control:**
- Additive Increase / Multiplicative Decrease limiter
- Success (2xx) → limit += 1
- Throttle (429) or Unavailable (503) → limit *= 0.5
- Configurable min/max limits
- FIFO queue for waiters when at capacity

**Retry Policy:**
- Only retries safe methods (GET, HEAD, OPTIONS)
- 429 → constant backoff + different node
- 503 → exponential backoff + different node
- 308 → permanent redirect to Location header
- 307 → temporary redirect
- Configurable max retries

**Node Selection:**
- Round-robin load balancing across URIs
- Health tracking with automatic recovery
- Unhealthy nodes retried after 30s recovery interval
- Least-recently-failed fallback

**Safety & Logging:**
- `SafeArg` / `UnsafeArg` type markers flow from DSL
- structlog processor respects annotations
- Unsafe args redacted → `"<REDACTED>"` in production logs

**Distributed Tracing:**
- W3C traceparent header propagation
- Optional OpenTelemetry integration
- Falls back to synthetic trace IDs

**Metrics:**
- Request counts (total, successful, failed, retried)
- Duration tracking (total, average)

#### Configuration

```python
config = ChannelConfig(
    uris=["https://api-1.example.com", "https://api-2.example.com"],
    connect_timeout=5.0,
    read_timeout=30.0,
    write_timeout=30.0,
    max_retries=3,
    initial_concurrency_limit=20,
    min_concurrency_limit=1,
    auth_token="Bearer ...",                     # Static token
    auth_token_provider=lambda: get_token(),     # Dynamic token
    user_agent="apollo-client/1.0",
    enable_http2=True,
    ca_bundle="/path/to/ca.pem",                 # TLS CA cert
    client_cert="/path/to/cert.pem",             # Client cert
    client_key="/path/to/key.pem",               # Client key
    verify_ssl=True,
    uri_file="/path/to/uris.txt",                # Live config reload
)

channel = EnchantChannel(config)
```

### 4. enchant-verify (Conformance Testing)

Proves generated code correctly implements the wire spec.

**Test types:**
- **Round-trip** — deserialize JSON → serialize → compare to original
- **Serialisation** — Python expr → serialize → compare to expected JSON
- **Deserialisation** — JSON → deserialize → validate structure
- **Validation** — validators correctly enforce constraints
- **Error handling** — error wire format parsed correctly
- **Optional handling** — null/missing fields handled correctly

```python
from enchant_verify import TestSuite, run_suite

suite = TestSuite.from_yaml("tests/conformance.yml")
results = run_suite(suite, type_registry=my_types)
print(f"{results.passed}/{results.total} passed")
```

### 5. enchant-cli (Orchestrator)

Click-based CLI that drives the full pipeline.

```bash
# Compile + generate all outputs
enchant build service.enchant.yml --models --server --client --output generated/

# Validate without generating
enchant check service.enchant.yml

# Run conformance tests
enchant verify service.enchant.yml --test-suite tests/conformance.yml

# Export OpenAPI for debugging/Swagger
enchant docs service.enchant.yml --output openapi.json

# Dump internal representation
enchant dump-ir service.enchant.yml

# Save snapshot for breaking change detection
enchant snapshot service.enchant.yml --output snapshots/

# Dry run (preview without writing)
enchant build service.enchant.yml --dry-run
```

### 6. pants-enchant (Build Plugin)

Pants build system plugin for CI/CD integration. Auto-discovers `.enchant.yml` files and generates code as part of the build graph.

---

## DSL Reference

### Type System

```yaml
types:
  # Primitives
  # string, integer, safelong, double, boolean, datetime, uuid, binary

  # Objects
  SearchRequest:
    fields:
      origin:
        type: string
        safety: safe
        docs: "IATA airport code"
        validation:
          min-length: 3
          max-length: 3
          pattern: "^[A-Z]{3}$"
      departure_date:
        type: datetime
        safety: safe
      passenger_name:
        type: string
        safety: unsafe        # Redacted in logs

  # Enums
  FlightClass:
    values:
      - ECONOMY
      - BUSINESS
      - FIRST

  # Unions (tagged discriminated)
  PaymentMethod:
    union:
      credit_card: CreditCardPayment
      bank_transfer: BankTransferPayment

  # Aliases (transparent wrappers with validation)
  AirportCode:
    alias: string
    validation:
      min-length: 3
      max-length: 3
      pattern: "^[A-Z]{3}$"

  # Containers
  # optional<T>, list<T>, set<T>, map<K,V>
  SearchResults:
    fields:
      flights: { type: "list<FlightResult>" }
      metadata: { type: "optional<SearchMetadata>" }
      tags: { type: "set<string>" }
      prices: { type: "map<string,double>" }
```

### Validation Keywords

| Keyword | Applies To | Example |
|---------|-----------|---------|
| `min-length` | string | `min-length: 1` |
| `max-length` | string | `max-length: 255` |
| `pattern` | string | `pattern: "^[A-Z]{3}$"` |
| `minimum` | integer, double | `minimum: 0` |
| `maximum` | integer, double | `maximum: 100` |
| `format` | string | `format: email` / `uri` / `ipv4` / `ipv6` |

### Safety Annotations

| Level | Behavior |
|-------|----------|
| `safe` | Logged normally in all environments |
| `unsafe` | Redacted to `<REDACTED>` in production logs |
| `do-not-log` | Never appears in any log output |

### Services

```yaml
services:
  FlightSearchService:
    base-path: /api/flights
    default-auth:
      type: header
      header: Authorization
    endpoints:
      searchFlights:
        http: GET /search
        args:
          origin: { type: string, param-type: query }
          destination: { type: string, param-type: query }
          departure_date: { type: datetime, param-type: query }
        returns: list<FlightResult>
        errors:
          - InvalidSearchCriteria

      getFlight:
        http: GET /{flightId}
        args:
          flightId: { type: string, param-type: path }
        returns: FlightResult
        errors:
          - FlightNotFound

      bookFlight:
        http: POST /book
        args:
          request: { type: BookingRequest, param-type: body }
        returns: BookingConfirmation
        errors:
          - FlightNotFound
          - PaymentFailed
```

### Errors

```yaml
errors:
  FlightNotFound:
    code: NOT_FOUND
    safe-args:
      flightId: string
    unsafe-args: {}

  PaymentFailed:
    code: FAILED_PRECONDITION
    safe-args:
      reason: string
    unsafe-args:
      cardLastFour: string
```

### Cross-File Imports

```yaml
types:
  imports:
    SharedTypes:
      file: shared-types.enchant.yml

  # Reference imported types as SharedTypes.TypeName
  FlightResult:
    fields:
      airport: { type: SharedTypes.Airport }
```

---

## Compiler Validation Rules

The spec compiler enforces 9 validation rules at compile time:

| Rule | What It Checks |
|------|---------------|
| **UniqueNames** | No duplicate type/service/error names across files |
| **NoRecursiveTypes** | Direct type recursion forbidden |
| **NoNestedOptionals** | `optional<optional<T>>` is illegal |
| **FieldCasing** | Fields must be camelCase, types PascalCase, enum values UPPER_SNAKE_CASE |
| **EndpointConstraints** | Path/query/header params must be primitives, enums, or aliases |
| **PathParamPresence** | All `{param}` in URL template must have matching arg |
| **SingleBody** | At most one body param per endpoint |
| **NoBodyOnGet** | GET/DELETE cannot have body params |
| **ValidationTargets** | Validation keywords must match field types |

---

## Code Examples for Apollo

### Apollo Service Definition

```yaml
# apollo-catalog.enchant.yml
types:
  Product:
    fields:
      id: { type: integer, safety: safe }
      groupId: { type: string, safety: safe, validation: { min-length: 1 } }
      artifactId: { type: string, safety: safe, validation: { min-length: 1 } }
      name: { type: string, safety: safe }
      description: { type: "optional<string>", safety: safe }
      productType: { type: string, safety: safe }
      ownerTeam: { type: "optional<string>", safety: safe }

  Release:
    fields:
      id: { type: integer, safety: safe }
      version: { type: string, safety: safe }
      releaseType: { type: ReleaseType, safety: safe }
      productId: { type: integer, safety: safe }
      isRecalled: { type: boolean, safety: safe }

  ReleaseType:
    values:
      - RELEASE
      - RELEASE_CANDIDATE
      - SNAPSHOT
      - RC_SNAPSHOT

  EntityState:
    values:
      - UNMANAGED
      - MANAGED
      - INSTALLING
      - RUNNING
      - UPDATING
      - FAILED
      - STOPPED

  Entity:
    fields:
      id: { type: integer, safety: safe }
      name: { type: string, safety: safe, validation: { min-length: 1, max-length: 253 } }
      environmentId: { type: integer, safety: safe }
      state: { type: EntityState, safety: safe }
      configOverrides: { type: "map<string,string>", safety: safe }
      namespace: { type: "optional<string>", safety: safe }

  CreateProductRequest:
    fields:
      groupId: { type: string, safety: safe, validation: { min-length: 1 } }
      artifactId: { type: string, safety: safe, validation: { min-length: 1 } }
      name: { type: string, safety: safe }
      description: { type: "optional<string>", safety: safe }

errors:
  ProductNotFound:
    code: NOT_FOUND
    safe-args:
      productId: string

  InvalidProductDefinition:
    code: INVALID_ARGUMENT
    safe-args:
      reason: string

services:
  CatalogService:
    base-path: /api/catalog
    default-auth:
      type: header
      header: Authorization
    endpoints:
      listProducts:
        http: GET /products
        returns: list<Product>

      getProduct:
        http: GET /products/{productId}
        args:
          productId: { type: integer, param-type: path }
        returns: Product
        errors:
          - ProductNotFound

      createProduct:
        http: POST /products
        args:
          request: { type: CreateProductRequest, param-type: body }
        returns: Product
        errors:
          - InvalidProductDefinition

      getReleasesForProduct:
        http: GET /products/{productId}/releases
        args:
          productId: { type: integer, param-type: path }
        returns: list<Release>
        errors:
          - ProductNotFound
```

### Building Generated Code

```bash
# Generate all outputs
enchant build apollo-catalog.enchant.yml \
  --models --server --client \
  --output apollo/generated/

# Produces:
# apollo/generated/models/catalog_types.py     (attrs classes)
# apollo/generated/server/catalog_service.py    (abstract base + router)
# apollo/generated/client/catalog_client.py     (httpx client)
```

### Implementing Server Stub

```python
# apollo/services/catalog_impl.py
"""Implement the generated CatalogService abstract base."""

from apollo.generated.server.catalog_service import CatalogServiceBase
from apollo.generated.models.catalog_types import (
    Product, Release, CreateProductRequest,
)
from apollo.catalog.crud import (
    list_products, get_product_by_id, create_product, get_releases_for_product,
)


class CatalogServiceImpl(CatalogServiceBase):
    def __init__(self, session_factory):
        self._session_factory = session_factory

    async def list_products(self) -> list[Product]:
        with self._session_factory() as session:
            rows = list_products(session)
            return [self._row_to_product(r) for r in rows]

    async def get_product(self, product_id: int) -> Product:
        with self._session_factory() as session:
            row = get_product_by_id(session, product_id)
            if not row:
                raise ProductNotFound(product_id=str(product_id))
            return self._row_to_product(row)

    async def create_product(self, request: CreateProductRequest) -> Product:
        with self._session_factory() as session:
            row = create_product(
                session,
                group_id=request.group_id,
                artifact_id=request.artifact_id,
                name=request.name,
                description=request.description,
            )
            return self._row_to_product(row)

    async def get_releases_for_product(self, product_id: int) -> list[Release]:
        with self._session_factory() as session:
            rows = get_releases_for_product(session, product_id)
            return [self._row_to_release(r) for r in rows]
```

### Using Generated Client

```python
# apollo/spoke/agent/catalog_client.py
"""Use the generated CatalogService client."""

from enchant_dialogue import EnchantChannel, ChannelConfig
from apollo.generated.client.catalog_client import CatalogServiceClient

config = ChannelConfig(
    uris=["https://hub.apollo.internal:8443"],
    auth_token_provider=lambda: get_service_token(),
    max_retries=3,
    connect_timeout=5.0,
    read_timeout=30.0,
)

channel = EnchantChannel(config)
client = CatalogServiceClient(channel)

# Type-safe, resilient API calls
products = client.list_products()
product = client.get_product(product_id=42)
releases = client.get_releases_for_product(product_id=42)
```

---

## Design Principles

1. **DSL is the only authoring surface** — OpenAPI 3.1 is internal, never exposed to developers
2. **Constraint enables quality** — No inheritance, no defaults, no nested optionals, limited type system
3. **Generated code is sacred** — Developers implement interfaces, never modify generated code
4. **Safety is type-level** — Safe/unsafe flows from DSL → models → logging → wire
5. **Forward compatibility by default** — Clients tolerate unknown fields (enum UNKNOWN, unknown union variants)
6. **Verification as first-class** — enchant-verify proves the pipeline is correct

---

## Integration with Witchcraft Ecosystem

```
enchant (code generation)
    ├── Generated models use attrs + cattrs (same as rune)
    ├── Generated clients use enchant-dialogue (httpx + AIMD + retry)
    ├── Generated servers implement witchcraft-cherry interfaces
    └── Safety annotations flow through structlog

rune (validation/serialization)
    ├── Same attrs + cattrs foundation as enchant models
    ├── Constraints can augment generated models
    └── Converter handles serialization for both

transmute (migrations)
    ├── Migrates schemas that enchant models depend on
    ├── Same attrs + cattrs for migration source/target schemas
    └── Dual-write proxy works with generated server stubs

sqlalchemy 2.0 (persistence)
    ├── ORM tables map to/from generated attrs models
    └── Transmute manages schema evolution
```

---

## Best Practices

### 1. One Definition File per Service Domain

```
apollo/
├── definitions/
│   ├── catalog.enchant.yml      # Product, Release, ReleaseChannel
│   ├── orchestration.enchant.yml # Plan, Constraint
│   ├── environment.enchant.yml   # Environment, Entity
│   └── shared-types.enchant.yml  # Common types (imported by others)
```

### 2. Never Modify Generated Code

Generated code is overwritten on every build. Put implementation in separate files that import from generated modules.

### 3. Use Safety Annotations Consistently

Mark every field with its safety level. PII is always `unsafe`. Internal IDs are `safe`. Secrets are `do-not-log`.

### 4. Validate at the DSL Level

Push validation into the `.enchant.yml` definition (min-length, pattern, format). The generated code enforces it automatically.

### 5. Use Conformance Tests in CI

Run `enchant verify` in CI to catch serialization regressions:

```bash
enchant verify definitions/*.enchant.yml --test-suite tests/conformance/
```

---

## References

- [attrs Documentation](https://www.attrs.org/)
- [cattrs Documentation](https://catt.rs/)
- [httpx Documentation](https://www.python-httpx.org/)
- [OpenAPI 3.1 Specification](https://spec.openapis.org/oas/v3.1.0)
- [structlog Documentation](https://www.structlog.org/)
