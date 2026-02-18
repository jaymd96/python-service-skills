# Rune — Composable Validation & Serialization for Apollo-on-k3s

## Overview

**Rune** is a composable validation, serialization, and JSON Schema generation library built on **attrs** and **cattrs**. It provides Pydantic-like capabilities — constraints, validators, aliases, computed fields, error accumulation — without requiring a base class. Validation is opt-in and explicit, keeping normal construction fast.

| Property | Value |
|----------|-------|
| **Package** | `rune` |
| **Version** | 0.1.0 |
| **License** | MIT |
| **Python** | >=3.11 |
| **Dependencies** | `attrs>=23.2`, `cattrs>=24.1` |
| **Optional** | `orjson>=3.9` (fast JSON) |
| **Repository** | witchcraft/rune |

### Why Rune for Apollo-on-k3s

| Capability | Apollo Use Case |
|------------|-----------------|
| Opt-in validation via `.parse()` | Validate at API boundaries, fast internal construction |
| Annotated constraints (`Gt`, `MinLen`, `Pattern`) | Plan constraints, version ranges, K8s name validation |
| Three-tier aliases | camelCase API ↔ snake_case internal ↔ DB column names |
| Computed fields | Derived properties in API responses (e.g., `product_id`) |
| JSON Schema generation | OpenAPI documentation, external tool integration |
| Custom validators | SemVer validation, dependency compatibility, config merging |
| TypeRegistry | Custom types (Version, MavenCoordinate, K8sName) |
| Unified with Transmute | Same attrs/cattrs foundation as migration framework |

### Installation

```bash
pip install rune
```

---

## Project Integration

### Location in Project

```
apollo/
├── models/
│   ├── __init__.py
│   ├── base.py                 # Base configurations, reusable annotated types
│   ├── environment.py          # Environment, Hub/Spoke models
│   ├── entity.py               # Entity state and configuration
│   ├── product.py              # Product, Release, Version models
│   ├── module.py               # Module and CompositeModule definitions
│   ├── plan.py                 # Plan, Constraint models
│   ├── channel.py              # Release channel definitions
│   ├── agent.py                # Agent state and commands
│   └── api/
│       ├── __init__.py
│       ├── requests.py         # API request schemas
│       └── responses.py        # API response schemas
├── validators/
│   └── constraints.py          # Reusable constraint types and validators
└── config/
    └── settings.py             # Environment-based configuration
```

### How We'll Use It

1. **Domain Models** — `@attrs.define` classes with Rune constraints for all Apollo data structures
2. **API Boundary Validation** — `.parse()` at API entry points, fast construction internally
3. **Serialization** — `Converter` with mode="json" for API responses, mode="python" for internal use
4. **JSON Schema** — `json_schema()` for documentation and external tool integration
5. **Custom Types** — `TypeRegistry` for Apollo-specific types (SemVer, MavenCoordinate, K8sName)

---

## API Quick Reference (llms.txt style)

### Core Imports

```python
from rune import (
    # Field definition
    field,                    # Enhanced attrs.field() with aliases, exclude, schema metadata

    # Class decorator
    inscribed,                # Adds .parse() and .validate() classmethods

    # Parsing and validation
    parse,                    # Function: dict -> validated instance
    validate,                 # Function: validate existing instance

    # Constraints (use with Annotated)
    Gt, Ge, Lt, Le,           # Numeric: greater/less than (exclusive/inclusive)
    MultipleOf,               # Numeric: must be multiple of value
    MinLen, MaxLen,            # String/collection length bounds
    Pattern,                  # String regex pattern
    MinItems, MaxItems,       # Collection item count bounds
    UniqueItems,              # Collection uniqueness

    # Schema metadata (use with Annotated)
    Description,              # Field description
    Examples,                 # Example values
    Deprecated,               # Mark field deprecated
    Title,                    # Override field title

    # Validators
    field_validator,          # Decorator: per-field validation
    model_validator,          # Decorator: cross-field validation

    # Serialization
    Converter,                # Enhanced cattrs converter with aliases and modes
    computed,                 # Decorator: computed fields in serialization

    # Aliases
    camel_case_aliases,       # Class decorator: auto camelCase aliases
    to_camel,                 # snake_case -> camelCase
    to_pascal,                # snake_case -> PascalCase
    to_kebab,                 # snake_case -> kebab-case
    to_snake,                 # camelCase -> snake_case

    # Schema generation
    json_schema,              # Generate JSON Schema draft 2020-12

    # Type system
    TypeRegistry,             # Register custom type handlers
    default_registry,         # Pre-loaded: datetime, UUID, Decimal, Path, IP

    # Errors
    ValidationError,          # Accumulated validation errors
    ErrorDetail,              # Single error with loc, msg, type, input
    ErrorTypes,               # Error type constants

    # Coercion
    CoercionRegistry,         # Type coercion management

    # Sentinels
    MISSING,                  # Sentinel for missing values (distinct from None)
    UNSET,                    # Sentinel for explicitly unset values
)
```

### Model Definition

```python
import attrs
from typing import Annotated
from rune import inscribed, field, Gt, MinLen, MaxLen, Pattern

@inscribed
@attrs.define
class ModelName:
    """Plain attrs class with Rune validation."""

    # Required field with constraints
    name: Annotated[str, MinLen(1), MaxLen(100)]

    # Required field with numeric constraint
    age: Annotated[int, Gt(0)]

    # Required field with regex pattern
    code: Annotated[str, Pattern(r'^[a-z][a-z0-9-]*$')]

    # Optional field
    description: str | None = None

    # Field with default
    status: str = "pending"

    # Field with alias (input/output name differs from attr name)
    email: str = field(alias="emailAddress")

    # Field with separate input/output aliases
    k8s_ns: str = field(
        validation_alias="k8sNamespace",     # Accept this key on input
        serialisation_alias="namespace",      # Emit this key on output
    )

    # Field excluded from serialization
    internal_id: int = field(exclude=True)

    # Field with factory default
    tags: list[str] = attrs.Factory(list)
```

### Parsing (Validated Construction)

```python
# Parse from dict — validates constraints, aliases, types
user = ModelName.parse({
    "name": "Alice",
    "age": 30,
    "code": "alice-1",
    "emailAddress": "alice@example.com",
    "k8sNamespace": "default",
    "internal_id": 42,
})

# Parse with strict mode — rejects unexpected keys
user = ModelName.parse(data, strict=True)

# Parse with custom converter
converter = Converter(mode="json")
user = ModelName.parse(data, converter=converter)

# Parse without coercion (strict types)
user = ModelName.parse(data, coerce=False)

# Function form (without @inscribed)
from rune import parse
user = parse(data, ModelName)
```

### Fast Construction (No Validation)

```python
# Normal attrs construction — no validation, no coercion
user = ModelName(
    name="Alice",
    age=30,
    code="alice-1",
    email="alice@example.com",
    k8s_ns="default",
    internal_id=42,
)
```

### Validate Existing Instance

```python
from rune import validate

# Validate an already-constructed instance
validated = validate(user)

# Or via classmethod (with @inscribed)
validated = ModelName.validate(user)
```

### Constraints

```python
from typing import Annotated
from rune import Gt, Ge, Lt, Le, MultipleOf, MinLen, MaxLen, Pattern
from rune import MinItems, MaxItems, UniqueItems
from rune import Description, Examples, Title, Deprecated

# Numeric
age: Annotated[int, Gt(0), Lt(150)]                # 0 < age < 150
priority: Annotated[int, Ge(1), Le(10)]             # 1 <= priority <= 10
batch_size: Annotated[int, MultipleOf(100)]          # 100, 200, 300...

# String
name: Annotated[str, MinLen(1), MaxLen(63)]
k8s_name: Annotated[str, Pattern(r'^[a-z][a-z0-9-]*$')]

# Collections
tags: Annotated[list[str], MinItems(1), MaxItems(10)]
ids: Annotated[list[int], UniqueItems()]

# Schema metadata (no runtime validation effect)
port: Annotated[int, Gt(0), Le(65535), Description("TCP port number")]
version: Annotated[str, Examples(["1.0.0", "2.1.3"]), Title("SemVer")]
old_field: Annotated[str, Deprecated()]
```

### Validators

```python
import attrs
from rune import field_validator, model_validator

@attrs.define
class Plan:
    plan_type: str
    target_version: str | None = None
    start_time: datetime
    end_time: datetime

    # Field validator — runs after structuring
    @field_validator("target_version")
    @classmethod
    def validate_target(cls, v: str | None) -> str | None:
        if v is not None:
            from semantic_version import Version
            try:
                Version(v)
            except ValueError:
                raise ValueError(f'"{v}" is not valid semver')
        return v

    # Model validator — cross-field validation
    @model_validator
    @classmethod
    def validate_dates(cls, instance):
        if instance.end_time <= instance.start_time:
            raise ValueError("end_time must be after start_time")
        return instance
```

### Converter (Serialization)

```python
from rune import Converter

c = Converter()

# Structure: dict -> instance (with alias remapping)
user = c.structure({"emailAddress": "a@b.com", "name": "Alice"}, User)

# Unstructure: instance -> dict
d = c.unstructure(user)
# {"name": "Alice", "email": "a@b.com"}

# JSON mode: converts datetime, UUID, Decimal, etc. to JSON-safe types
d = c.unstructure(user, mode="json")

# Alias mode: use aliases as output keys
d = c.unstructure(user, by_alias=True)
# {"name": "Alice", "emailAddress": "a@b.com"}

# Both modes combined
d = c.unstructure(user, mode="json", by_alias=True)
```

**Converter Constructor:**

```python
Converter(
    mode="python",            # Default unstructure mode: "python" | "json"
    by_alias=False,           # Use aliases by default in unstructure
    type_registry=None,       # Custom TypeRegistry (uses default if None)
    forbid_extra_keys=False,  # Reject unknown keys during structure
)
```

**Mode Behavior:**

| Type | Python Mode | JSON Mode |
|------|-------------|-----------|
| `datetime` | `datetime` object | ISO 8601 string |
| `date` | `date` object | ISO string |
| `UUID` | `UUID` object | String |
| `Decimal` | `Decimal` object | String |
| `Path` | `Path` object | String |
| `Enum` | `Enum` member | Enum value |
| `set`/`frozenset` | `list` | `list` |
| `bytes` | `bytes` | UTF-8 string |
| `timedelta` | `timedelta` | Total seconds (float) |

### Computed Fields

```python
import attrs
from rune import computed, Converter

@attrs.define
class Invoice:
    quantity: int
    unit_price: float

    @computed
    def total(self) -> float:
        return self.quantity * self.unit_price

    @computed(alias="formattedTotal")
    def formatted(self) -> str:
        return f"${self.total:.2f}"

c = Converter()
d = c.unstructure(Invoice(quantity=3, unit_price=10.0))
# {"quantity": 3, "unit_price": 10.0, "total": 30.0, "formatted": "$30.00"}

d = c.unstructure(Invoice(quantity=3, unit_price=10.0), by_alias=True)
# {"quantity": 3, "unit_price": 10.0, "total": 30.0, "formattedTotal": "$30.00"}
```

### Case Conversion Aliases

```python
import attrs
from rune import camel_case_aliases, inscribed

@inscribed
@camel_case_aliases
@attrs.define
class User:
    first_name: str     # alias: "firstName"
    last_name: str      # alias: "lastName"
    email_address: str  # alias: "emailAddress"

# Accepts camelCase input
user = User.parse({"firstName": "Alice", "lastName": "Smith", "emailAddress": "a@b.com"})
```

**Available converters:**

```python
from rune import to_camel, to_pascal, to_kebab, to_snake

to_camel("user_name")     # "userName"
to_pascal("user_name")    # "UserName"
to_kebab("user_name")     # "user-name"
to_snake("userName")      # "user_name"
```

### Type Registry

```python
from rune import TypeRegistry, TypeHandler, default_registry

# Default registry includes: datetime, date, time, UUID, Decimal, Path, IPv4, IPv6

# Create custom registry extending defaults
registry = default_registry.copy()

# Register a custom type
from semantic_version import Version

registry.register(
    Version,
    structure_fn=lambda v, _: Version(v) if isinstance(v, str) else v,
    unstructure_fn=lambda v: str(v),
    schema_fn=lambda _: {"type": "string", "pattern": r"^\d+\.\d+\.\d+"},
    validate_fn=lambda v: None,  # No extra validation needed
)

# Use with Converter
c = Converter(type_registry=registry)
```

### Coercion Registry

```python
from rune import CoercionRegistry

# Default coercions (registered via CoercionRegistry.with_defaults()):
#   str -> int, float, Decimal, bool, datetime, date, time, UUID, Path
#   int -> float, Decimal, bool, str
#   float -> int (lossless), Decimal, str
#   bool -> int, str

# Custom coercion
registry = CoercionRegistry.with_defaults()
registry.register(str, Version, lambda v: Version(v))

# Use in parse
user = User.parse(data, coercion_registry=registry)
```

### Discriminated Unions

```python
import attrs
from typing import Literal, Union
from rune import Converter

@attrs.define
class CreditCard:
    type: Literal["credit_card"] = "credit_card"
    number: str

@attrs.define
class BankTransfer:
    type: Literal["bank"] = "bank"
    account: str

PaymentMethod = Union[CreditCard, BankTransfer]

c = Converter()
c.register_discriminated_union(PaymentMethod, discriminator="type")
# Auto-detects: {"credit_card": CreditCard, "bank": BankTransfer}

# Structure by discriminator
payment = c.structure({"type": "credit_card", "number": "4111..."}, PaymentMethod)
assert isinstance(payment, CreditCard)
```

### JSON Schema Generation

```python
from rune import json_schema

schema = json_schema(User)
# {
#   "type": "object",
#   "title": "User",
#   "properties": {
#     "name": {"type": "string", "minLength": 1, "maxLength": 100},
#     "age": {"type": "integer", "exclusiveMinimum": 0},
#     ...
#   },
#   "required": ["name", "age", ...]
# }

# With aliases as property names
schema = json_schema(User, by_alias=True)

# Serialization mode (includes computed fields)
schema = json_schema(User, mode="serialisation")

# Custom title and description
schema = json_schema(User, title="UserModel", description="A user record")
```

**Type → JSON Schema Mapping:**

| Python Type | JSON Schema |
|-------------|-------------|
| `str` | `{"type": "string"}` |
| `int` | `{"type": "integer"}` |
| `float` | `{"type": "number"}` |
| `bool` | `{"type": "boolean"}` |
| `None` | `{"type": "null"}` |
| `bytes` | `{"type": "string", "contentEncoding": "base64"}` |
| `datetime` | `{"type": "string", "format": "date-time"}` |
| `date` | `{"type": "string", "format": "date"}` |
| `UUID` | `{"type": "string", "format": "uuid"}` |
| `Decimal` | `{"type": "string", "format": "decimal"}` |
| `Path` | `{"type": "string", "format": "path"}` |
| `list[X]` | `{"type": "array", "items": X_schema}` |
| `dict[K, V]` | `{"type": "object", "additionalProperties": V_schema}` |
| `set[X]` | `{"type": "array", "uniqueItems": true, "items": X_schema}` |
| `Literal["a", "b"]` | `{"enum": ["a", "b"]}` |
| `Optional[X]` | `{"anyOf": [X_schema, {"type": "null"}]}` |
| `Enum` | `{"enum": [values]}` |
| Nested attrs | `{"$ref": "#/$defs/ClassName"}` |

### Error Handling

```python
from rune import ValidationError, ErrorDetail, ErrorTypes

try:
    user = User.parse({"name": "", "age": -5})
except ValidationError as e:
    e.error_count()          # 2
    e.errors()               # list of error dicts
    e.error_details()        # list of ErrorDetail instances
    e.json(indent=2)         # JSON string
    e.by_field()             # {("name",): [...], ("age",): [...]}

# Each error dict:
# {
#   "loc": ("name",),
#   "msg": "string length must be >= 1",
#   "type": "min_length",
#   "input": "",
#   "ctx": {"min_length": 1}
# }
```

**Error Type Constants:**

```python
ErrorTypes.MISSING                 # "missing"
ErrorTypes.UNEXPECTED              # "unexpected_field"
ErrorTypes.TYPE_ERROR              # "type_error"
ErrorTypes.TYPE_ERROR_INT          # "type_error.integer"
ErrorTypes.GREATER_THAN            # "greater_than"
ErrorTypes.LESS_THAN               # "less_than"
ErrorTypes.MIN_LENGTH              # "min_length"
ErrorTypes.MAX_LENGTH              # "max_length"
ErrorTypes.PATTERN                 # "pattern_mismatch"
ErrorTypes.MIN_ITEMS               # "min_items"
ErrorTypes.UNIQUE_ITEMS            # "unique_items"
ErrorTypes.VALUE_ERROR             # "value_error"
```

### Sentinels

```python
from rune import MISSING, UNSET

# MISSING — a required value was not provided (distinct from None)
# UNSET — a value was explicitly unset (for partial updates)

# Both are falsy:
assert not MISSING
assert not UNSET

# But distinct from None:
assert MISSING is not None
assert UNSET is not None
assert MISSING is not UNSET
```

---

## Code Examples for Apollo

### Base Model with Common Configuration

```python
# apollo/models/base.py
"""Base model definitions and reusable annotated types for Apollo."""

import attrs
from datetime import datetime
from typing import Annotated
from uuid import UUID, uuid4

from rune import (
    inscribed, field, computed,
    MinLen, MaxLen, Pattern, Gt, Le,
    Description,
    field_validator,
    TypeRegistry, default_registry,
)
from semantic_version import Version


# ---------------------------------------------------------------------------
# Custom Type Registry
# ---------------------------------------------------------------------------

apollo_registry = default_registry.copy()

apollo_registry.register(
    Version,
    structure_fn=lambda v, _: Version(v) if isinstance(v, str) else v,
    unstructure_fn=lambda v: str(v),
    schema_fn=lambda _: {"type": "string", "pattern": r"^\d+\.\d+\.\d+"},
)


# ---------------------------------------------------------------------------
# Reusable Annotated Types
# ---------------------------------------------------------------------------

K8sName = Annotated[
    str,
    MinLen(1), MaxLen(63),
    Pattern(r'^[a-z][a-z0-9-]*[a-z0-9]$'),
    Description("Valid Kubernetes resource name"),
]

MavenCoordinate = Annotated[
    str,
    Pattern(r'^[a-z0-9.-]+:[a-z0-9.-]+$'),
    Description("Maven coordinate: groupId:artifactId"),
]

SemVer = Annotated[
    str,
    Pattern(r'^\d+\.\d+\.\d+'),
    Description("Semantic version string"),
]

K8sLabels = Annotated[
    dict[str, str],
    Description("Kubernetes-style labels (key/value <= 63 chars)"),
]


# ---------------------------------------------------------------------------
# Base Models
# ---------------------------------------------------------------------------

@inscribed
@attrs.define
class TimestampedModel:
    """Base for models with creation/modification tracking."""
    created_at: datetime = attrs.Factory(datetime.utcnow)
    updated_at: datetime = attrs.Factory(datetime.utcnow)


@inscribed
@attrs.define
class IdentifiableModel(TimestampedModel):
    """Base for models with UUID identifiers."""
    id: UUID = attrs.Factory(uuid4)
```

### Environment Model

```python
# apollo/models/environment.py
"""Environment models for Apollo hub/spoke architecture."""

import attrs
from enum import Enum
from typing import Any

from rune import inscribed, field, field_validator

from .base import IdentifiableModel, K8sName, K8sLabels


class EnvironmentType(str, Enum):
    HUB = "hub"
    SPOKE = "spoke"


@inscribed
@attrs.define
class Environment(IdentifiableModel):
    """Represents an Apollo Environment (Hub or Spoke)."""

    name: K8sName
    display_name: str = field(serialisation_alias="displayName")
    environment_type: EnvironmentType
    description: str | None = None
    labels: K8sLabels = attrs.Factory(dict)
    config: dict[str, Any] = attrs.Factory(dict)

    # Hub-specific
    managed_spokes: list[str] | None = None

    # Spoke-specific
    hub_id: str | None = None

    @field_validator("name")
    @classmethod
    def validate_k8s_name(cls, v: str) -> str:
        if v.startswith("-") or v.endswith("-"):
            raise ValueError("name cannot start or end with hyphen")
        return v

    @field_validator("labels")
    @classmethod
    def validate_labels(cls, v: dict[str, str]) -> dict[str, str]:
        for key, value in v.items():
            if len(key) > 63 or len(value) > 63:
                raise ValueError("label keys and values must be <= 63 chars")
        return v
```

### Entity Model with State

```python
# apollo/models/entity.py
"""Entity models for deployed product instances."""

import attrs
from enum import Enum
from typing import Annotated, Any

from rune import inscribed, field, model_validator, MinLen, MaxLen, Pattern

from .base import IdentifiableModel, MavenCoordinate


class EntityState(str, Enum):
    UNMANAGED = "unmanaged"
    PENDING = "pending"
    INSTALLING = "installing"
    RUNNING = "running"
    DEGRADED = "degraded"
    FAILED = "failed"


class EntityHealth(str, Enum):
    HEALTHY = "healthy"
    UNHEALTHY = "unhealthy"
    UNKNOWN = "unknown"


@inscribed
@attrs.define
class Entity(IdentifiableModel):
    """An instance of a Product deployed to an Environment."""

    name: Annotated[str, MinLen(1), MaxLen(253)]
    environment_id: str
    product_id: MavenCoordinate
    namespace: Annotated[str, Pattern(r'^[a-z][a-z0-9-]*$')]

    # State
    state: EntityState = EntityState.UNMANAGED
    health: EntityHealth = EntityHealth.UNKNOWN
    current_version: str | None = None
    target_version: str | None = None

    # Configuration
    config_overrides: dict[str, Any] = attrs.Factory(dict)
    release_channel: str | None = None

    # Dependencies
    dependencies: list[str] = attrs.Factory(list)
    ignored_dependencies: list[str] = attrs.Factory(list)

    @model_validator
    @classmethod
    def validate_version_consistency(cls, instance):
        if instance.state == EntityState.RUNNING and not instance.current_version:
            raise ValueError("running entity must have current_version")
        return instance
```

### Plan and Constraint Models

```python
# apollo/models/plan.py
"""Plan and constraint models for Apollo orchestration."""

import attrs
from datetime import datetime
from enum import Enum
from typing import Annotated, Any, Literal, Union

from rune import inscribed, field, model_validator, Gt, Le, Converter

from .base import IdentifiableModel


class PlanState(str, Enum):
    PENDING = "pending"
    BLOCKED = "blocked"
    EXECUTING = "executing"
    SUCCEEDED = "succeeded"
    FAILED = "failed"
    ROLLED_BACK = "rolled_back"


class PlanType(str, Enum):
    INSTALL = "install"
    UPGRADE = "upgrade"
    CONFIG_UPDATE = "config_update"
    UNINSTALL = "uninstall"
    ROLLBACK = "rollback"


# --- Constraint Models (Discriminated Union) ---

@attrs.define(frozen=True)
class MaintenanceWindowConstraint:
    type: Literal["maintenance_window"] = "maintenance_window"
    cron_expression: str = ""
    duration_minutes: Annotated[int, Gt(0), Le(1440)] = 60
    timezone: str = "UTC"
    allows_downtime: bool = False


@attrs.define(frozen=True)
class DependencyConstraint:
    type: Literal["dependency"] = "dependency"
    depends_on_product: str = ""
    version_range: str = ""


@attrs.define(frozen=True)
class SuppressionConstraint:
    type: Literal["suppression"] = "suppression"
    until: datetime = attrs.Factory(datetime.utcnow)
    reason: str = ""
    auto_created: bool = False


Constraint = Union[MaintenanceWindowConstraint, DependencyConstraint, SuppressionConstraint]

# Register discriminated union
_converter = Converter()
_converter.register_discriminated_union(Constraint, discriminator="type")


# --- Plan Model ---

@inscribed
@attrs.define
class Plan(IdentifiableModel):
    """A unit of work to change an Entity's state."""

    entity_id: str = ""
    environment_id: str = ""
    plan_type: PlanType = PlanType.INSTALL
    state: PlanState = PlanState.PENDING

    # Target state
    target_version: str | None = None
    config_changes: dict[str, Any] | None = None

    # Constraints
    constraints: list[Constraint] = attrs.Factory(list)
    blocking_constraints: list[str] = attrs.Factory(list)

    # Execution tracking
    started_at: datetime | None = None
    completed_at: datetime | None = None
    error_message: str | None = None

    # Rollback reference
    rollback_of: str | None = None

    @model_validator
    @classmethod
    def validate_plan_requirements(cls, instance):
        if instance.plan_type in (PlanType.INSTALL, PlanType.UPGRADE):
            if not instance.target_version:
                raise ValueError(
                    f"{instance.plan_type.value} plan requires target_version"
                )
        return instance
```

### Module Definition Model

```python
# apollo/models/module.py
"""Module definitions for grouping products."""

import attrs
import re
from typing import Annotated, Any, Literal

from rune import inscribed, field, model_validator, MinLen, MaxLen, Pattern

from .base import K8sName, MavenCoordinate


@attrs.define
class ModuleVariable:
    """A variable that can be set during module installation."""

    name: Annotated[str, MinLen(1), MaxLen(63)]
    description: str | None = None
    default_value: str | None = None
    required: bool = True

    constraint_type: Literal["regex", "one_of", "none"] = "none"
    constraint_value: str | list[str] | None = None

    @model_validator
    @classmethod
    def validate_constraint(cls, instance):
        if instance.constraint_type == "regex" and instance.constraint_value:
            try:
                re.compile(str(instance.constraint_value))
            except re.error as e:
                raise ValueError(f"invalid regex pattern: {e}")
        return instance


@attrs.define
class ModuleEntity:
    """An entity definition within a Module."""

    type: Literal["helmChart"] = "helmChart"
    product_id: MavenCoordinate = field(validation_alias="productId")
    k8s_namespace: str = field(validation_alias="k8sNamespace")
    entity_name: str | None = None
    config_overrides: dict[str, Any] = attrs.Factory(dict)
    ignored_dependencies: list[str] = attrs.Factory(list)
    marked_for_uninstallation: bool = False
    secret_requirements: dict[str, Any] = attrs.Factory(dict)


@inscribed
@attrs.define
class Module:
    """A grouping of Products installed together."""

    name: K8sName
    display_name: str = field(validation_alias="displayName")
    description: str | None = None
    version: str = "0.1.0"

    entities: list[ModuleEntity] = attrs.Factory(list)
    variables: list[ModuleVariable] = attrs.Factory(list)

    allow_new_installations: bool = True
    allow_entity_overrides: bool = True
```

### API Request/Response Models

```python
# apollo/models/api/requests.py
"""API request schemas with strict validation."""

import attrs
from typing import Any

from rune import inscribed, field, MinLen, MaxLen, Pattern

from ..base import K8sName


@inscribed
@attrs.define(frozen=True)
class CreateEnvironmentRequest:
    name: K8sName
    display_name: str = field(
        validation_alias="displayName",
        serialisation_alias="displayName",
    )
    environment_type: str = ""
    description: str | None = None
    labels: dict[str, str] = attrs.Factory(dict)
    hub_id: str | None = None


@inscribed
@attrs.define(frozen=True)
class InstallModuleRequest:
    module_name: str = ""
    module_version: str = ""
    environment_id: str = ""
    variable_values: dict[str, str] = attrs.Factory(dict)
    entity_overrides: dict[str, Any] = attrs.Factory(dict)


# apollo/models/api/responses.py
"""API response wrappers."""

import attrs
from typing import Generic, TypeVar

T = TypeVar("T")


@attrs.define
class PaginationMeta:
    total: int = 0
    page: int = 1
    page_size: int = 20
    total_pages: int = 1


@attrs.define
class ApiResponse(Generic[T]):
    success: bool = True
    data: T | None = None
    message: str | None = None


@attrs.define
class PaginatedResponse(Generic[T]):
    success: bool = True
    data: list[T] = attrs.Factory(list)
    pagination: PaginationMeta = attrs.Factory(PaginationMeta)


@attrs.define
class ErrorResponse:
    success: bool = False
    error: str = ""
    error_code: str = ""
    details: dict[str, Any] | None = None
```

### Reusable Validators

```python
# apollo/validators/constraints.py
"""Reusable validation functions and annotated types for Apollo."""

from typing import Annotated
from rune import field_validator, TypeRegistry, default_registry
from rune import MinLen, MaxLen, Pattern, Description
from semantic_version import Version, NpmSpec


def validate_semver(v: str) -> str:
    """Validate semantic version string."""
    try:
        Version(v)
    except ValueError:
        raise ValueError(f'invalid semantic version: {v}')
    return v


def validate_semver_range(v: str) -> str:
    """Validate semantic version range expression."""
    try:
        NpmSpec(v)
    except ValueError:
        raise ValueError(f'invalid version range: {v}')
    return v


def validate_maven_coordinate(v: str) -> str:
    """Validate Maven coordinate format."""
    parts = v.split(":")
    if len(parts) not in (2, 3):
        raise ValueError("must be groupId:artifactId or groupId:artifactId:version")
    return v


# Reusable annotated types with both constraints and validators
SemVer = Annotated[str, Description("Semantic version (MAJOR.MINOR.PATCH)")]
SemVerRange = Annotated[str, Description("SemVer range expression")]
K8sName = Annotated[
    str,
    MinLen(1), MaxLen(63),
    Pattern(r'^[a-z][a-z0-9-]*[a-z0-9]$'),
]
MavenCoordinate = Annotated[
    str,
    Pattern(r'^[a-z0-9.-]+:[a-z0-9.-]+$'),
]
```

---

## Best Practices

### 1. Validate at Boundaries, Construct Freely Internally

```python
# API boundary — validate incoming data
entity = Entity.parse(request_data, strict=True)

# Internal code — fast, no validation
entity = Entity(
    name="control-plane",
    environment_id="prod-us-west",
    product_id="com.apollo:control-plane",
    namespace="apollo-system",
)

# From database — already validated, skip validation
entity = converter.structure(row_dict, Entity)
```

### 2. Use Annotated Types for Reusable Constraints

```python
# Define once
K8sName = Annotated[str, MinLen(1), MaxLen(63), Pattern(r'^[a-z][a-z0-9-]*$')]

# Use everywhere
@attrs.define
class Entity:
    name: K8sName
    namespace: K8sName
    helm_release: K8sName
```

### 3. Separate Validation and Serialization Aliases

```python
# Accept camelCase from API, emit snake_case to database, use Python names internally
@attrs.define
class Entity:
    k8s_namespace: str = field(
        validation_alias="k8sNamespace",       # API sends this
        serialisation_alias="k8s_namespace",   # DB column name
    )
```

### 4. Use Frozen Classes for Value Objects

```python
# Immutable constraints and config
@attrs.define(frozen=True)
class MaintenanceWindow:
    cron_expression: str
    duration_minutes: Annotated[int, Gt(0), Le(1440)]
    timezone: str = "UTC"
```

### 5. Leverage the Type Registry for Domain Types

```python
# Register once, use everywhere
apollo_registry.register(
    Version,
    structure_fn=lambda v, _: Version(v) if isinstance(v, str) else v,
    unstructure_fn=lambda v: str(v),
    schema_fn=lambda _: {"type": "string", "pattern": r"^\d+\.\d+\.\d+"},
)
```

### 6. Error Handling at API Boundaries

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

### 7. Use Computed Fields for Derived Data

```python
@attrs.define
class Product:
    group_id: str
    artifact_id: str

    @computed(alias="productId")
    def product_id(self) -> str:
        return f"{self.group_id}:{self.artifact_id}"
```

---

## Integration with Other Apollo Packages

### With SQLAlchemy 2.0 (Persistence)

```python
from rune import Converter

# ORM row -> domain model
converter = Converter()
entity = converter.structure(row.__dict__, Entity)

# Domain model -> ORM row update
data = converter.unstructure(entity)
for key, value in data.items():
    setattr(row, key, value)
```

### With Transmute (Migrations)

Rune and Transmute share the same attrs/cattrs foundation. Migration schemas can be Rune-decorated attrs classes:

```python
import attrs
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
    meta = MigrationMeta(...)
    source_schema = UserV1
    target_schema = UserV2

    def transform(self, record: UserV1) -> UserV2:
        first, last = record.full_name.split(" ", 1)
        return UserV2(id=record.id, first_name=first, last_name=last)
```

### With Transitions (State Machine)

```python
import attrs
from transitions import Machine

@attrs.define
class Entity:
    state: EntityState = EntityState.UNMANAGED
    _machine: Machine = attrs.field(init=False, repr=False, eq=False)

    def __attrs_post_init__(self):
        self._machine = Machine(
            model=self,
            states=[s.value for s in EntityState],
            initial=self.state.value,
        )
```

---

## References

- [attrs Documentation](https://www.attrs.org/)
- [cattrs Documentation](https://catt.rs/)
- [JSON Schema Draft 2020-12](https://json-schema.org/draft/2020-12/json-schema-core)
