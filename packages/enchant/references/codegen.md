# Enchant — Code Generation

> Part of the enchant skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [enchant-spec (DSL Compiler)](#enchant-spec)
- [gen-models (attrs Classes)](#gen-models)
- [gen-server (Abstract Base + Router)](#gen-server)
- [gen-client (Typed httpx Client)](#gen-client)
- [enchant-verify (Conformance Testing)](#enchant-verify)
- [enchant-cli (Orchestrator)](#enchant-cli)
- [Forward Compatibility](#forward-compatibility)

## enchant-spec

Compiles `.enchant.yml` YAML into OpenAPI 3.1 (internal representation).

```python
from enchant_spec import compile_to_openapi, compile_to_ast

openapi = compile_to_openapi(["service.enchant.yml", "shared-types.enchant.yml"])
definition = compile_to_ast(["service.enchant.yml"])  # For tooling
```

## gen-models

Generates frozen attrs classes with validators, aliases, and safety metadata:

```python
# Generated output
import attrs
from typing import Annotated
from enchant_runtime import SafeArg, UnsafeArg

@attrs.define(frozen=True, slots=True)
class SearchRequest:
    origin: Annotated[str, SafeArg] = attrs.field(
        metadata={"alias": "origin", "safety": "safe"}
    )
    passenger_name: Annotated[str, UnsafeArg] = attrs.field(
        metadata={"alias": "passengerName", "safety": "unsafe"}
    )
```

Also generates a cattrs Converter pre-configured with all type registrations.

## gen-server

Generates abstract base classes + registration function for Cherry router:

```python
# Generated output
from abc import ABC, abstractmethod

class FlightSearchServiceBase(ABC):
    @abstractmethod
    async def search_flights(self, origin: str, destination: str) -> list[FlightResult]:
        ...

    @abstractmethod
    async def get_flight(self, flight_id: str) -> FlightResult:
        ...

def register_flight_search_service(router, impl: FlightSearchServiceBase):
    """Wire implementation into HTTP router."""
    router.get("/api/flights/search", ...)
    router.get("/api/flights/{flightId}", ...)
    router.post("/api/flights/book", ...)
```

## gen-client

Generates typed client wrappers delegating to `EnchantChannel`:

```python
# Generated output
class FlightSearchServiceClient:
    def __init__(self, channel: EnchantChannel):
        self._channel = channel

    def search_flights(self, origin: str, destination: str) -> list[FlightResult]:
        return self._channel.execute(
            method="GET",
            path="/api/flights/search",
            params={"origin": origin, "destination": destination},
            response_type=list[FlightResult],
        )
```

## enchant-verify

Proves generated code correctly implements the wire spec.

```python
from enchant_verify import TestSuite, run_suite

suite = TestSuite.from_yaml("tests/conformance.yml")
results = run_suite(suite, type_registry=my_types)
print(f"{results.passed}/{results.total} passed")
```

### Test Types

| Type | What It Tests |
|------|--------------|
| **Round-trip** | deserialize JSON -> serialize -> compare to original |
| **Serialisation** | Python expr -> serialize -> compare to expected JSON |
| **Deserialisation** | JSON -> deserialize -> validate structure |
| **Validation** | Validators correctly enforce constraints |
| **Error handling** | Error wire format parsed correctly |
| **Optional handling** | null/missing fields handled correctly |

## enchant-cli

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

### pants-enchant

Pants build plugin for CI/CD integration. Auto-discovers `.enchant.yml` files and generates code as part of the build graph.

## Forward Compatibility

Generated clients tolerate unknown additions:
- **Unknown enum values**: Deserialized as `UNKNOWN` variant
- **Unknown union variants**: Deserialized as opaque wrapper
- **Unknown object fields**: Silently ignored during deserialization
