# Rune — Examples & Gotchas

> Part of the rune skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [Base Model with TypeRegistry](#base-model-with-typeregistry)
- [Reusable Annotated Types](#reusable-annotated-types)
- [API Boundary Validation](#api-boundary-validation)
- [Error Handling at Boundaries](#error-handling-at-boundaries)
- [Integration with SQLAlchemy](#integration-with-sqlalchemy)
- [Integration with Transmute](#integration-with-transmute)
- [Frozen Classes for Value Objects](#frozen-classes-for-value-objects)
- [Gotchas](#gotchas)

## Base Model with TypeRegistry

```python
from rune import TypeRegistry, default_registry, inscribed, field, computed
from semantic_version import Version
import attrs
from datetime import datetime
from uuid import UUID, uuid4

# Custom registry extending defaults
apollo_registry = default_registry.copy()
apollo_registry.register(
    Version,
    structure_fn=lambda v, _: Version(v) if isinstance(v, str) else v,
    unstructure_fn=lambda v: str(v),
    schema_fn=lambda _: {"type": "string", "pattern": r"^\d+\.\d+\.\d+"},
)

@inscribed
@attrs.define
class TimestampedModel:
    created_at: datetime = attrs.Factory(datetime.utcnow)
    updated_at: datetime = attrs.Factory(datetime.utcnow)

@inscribed
@attrs.define
class IdentifiableModel(TimestampedModel):
    id: UUID = attrs.Factory(uuid4)
```

## Reusable Annotated Types

```python
from typing import Annotated
from rune import MinLen, MaxLen, Pattern, Description

K8sName = Annotated[
    str, MinLen(1), MaxLen(63),
    Pattern(r'^[a-z][a-z0-9-]*[a-z0-9]$'),
    Description("Valid Kubernetes resource name"),
]

MavenCoordinate = Annotated[
    str, Pattern(r'^[a-z0-9.-]+:[a-z0-9.-]+$'),
    Description("Maven coordinate: groupId:artifactId"),
]

SemVer = Annotated[
    str, Pattern(r'^\d+\.\d+\.\d+'),
    Description("Semantic version string"),
]

# Use everywhere
@attrs.define
class Entity:
    name: K8sName
    namespace: K8sName
    product_id: MavenCoordinate
```

## API Boundary Validation

```python
# API boundary — validate incoming data
entity = Entity.parse(request_data, strict=True)

# Internal code — fast, no validation
entity = Entity(name="control-plane", namespace="apollo-system", ...)

# From database — already validated, skip validation
entity = converter.structure(row_dict, Entity)
```

## Error Handling at Boundaries

```python
from rune import ValidationError

try:
    request = CreateEnvironmentRequest.parse(data, strict=True)
except ValidationError as e:
    return ErrorResponse(
        error="Validation failed",
        error_code="VALIDATION_ERROR",
        details={"errors": e.errors()},
    )
```

## Integration with SQLAlchemy

```python
from rune import Converter

converter = Converter()

# ORM row -> domain model
entity = converter.structure(row.__dict__, Entity)

# Domain model -> ORM row update
data = converter.unstructure(entity)
for key, value in data.items():
    setattr(row, key, value)
```

## Integration with Transmute

Rune and Transmute share the same attrs/cattrs foundation:

```python
import attrs
from typing import Annotated
from rune import inscribed, MinLen
from transmute import Migration, MigrationMeta

@inscribed
@attrs.define
class UserV1:
    id: str
    full_name: Annotated[str, MinLen(1)]

@inscribed
@attrs.define
class UserV2:
    id: str
    first_name: Annotated[str, MinLen(1)]
    last_name: Annotated[str, MinLen(1)]

class SplitName(Migration[UserV1, UserV2]):
    def transform(self, record: UserV1) -> UserV2:
        first, last = record.full_name.split(" ", 1)
        return UserV2(id=record.id, first_name=first, last_name=last)
```

## Frozen Classes for Value Objects

```python
@attrs.define(frozen=True)
class MaintenanceWindow:
    cron_expression: str
    duration_minutes: Annotated[int, Gt(0), Le(1440)]
    timezone: str = "UTC"
```

## Gotchas

1. **Decorator order matters**: `@inscribed` goes ABOVE `@attrs.define`
2. **`.parse()` validates, constructor does NOT**: `User(age=-1)` succeeds silently, `User.parse({"age": -1})` raises `ValidationError`
3. **Mutable defaults**: Use `attrs.Factory(list)`, never `= []`
4. **Constraints need Annotated**: `age: int` has no constraints; `age: Annotated[int, Gt(0)]` does
5. **Aliases don't affect constructor**: `User(email_addr="...")` not `User(emailAddress="...")`
6. **Converter modes are per-call**: `c.unstructure(x, mode="json")` overrides the constructor default
