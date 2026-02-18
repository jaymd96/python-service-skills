# Rune — Core API Reference

> Part of the rune skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [Core Imports](#core-imports)
- [Model Definition](#model-definition)
- [Parsing and Validation](#parsing-and-validation)
- [Constraints](#constraints)
- [Validators](#validators)
- [Converter](#converter)
- [TypeRegistry](#typeregistry)
- [CoercionRegistry](#coercionregistry)
- [Error Handling](#error-handling)
- [Sentinels](#sentinels)

## Core Imports

```python
from rune import (
    # Class decorator
    inscribed,                # Adds .parse() and .validate() classmethods

    # Field definition
    field,                    # Enhanced attrs.field() with aliases, exclude, schema metadata

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

    # Schema metadata (use with Annotated — no runtime validation effect)
    Description,              # Field description
    Examples,                 # Example values
    Title,                    # Override field title
    Deprecated,               # Mark field deprecated

    # Validators
    field_validator,          # Decorator: per-field validation
    model_validator,          # Decorator: cross-field validation

    # Serialization
    Converter,                # Enhanced cattrs converter with aliases and modes
    computed,                 # Decorator: computed fields in serialization

    # Aliases
    camel_case_aliases,       # Class decorator: auto camelCase aliases
    to_camel, to_pascal, to_kebab, to_snake,  # Case conversion helpers

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

## Model Definition

`@inscribed` adds `.parse()` and `.validate()` classmethods to an attrs class. Place **above** `@attrs.define`:

```python
import attrs
from typing import Annotated
from rune import inscribed, field, Gt, MinLen, Pattern

@inscribed
@attrs.define
class User:
    name: Annotated[str, MinLen(1), MaxLen(100)]
    age: Annotated[int, Gt(0)]
    email: str | None = None
    status: str = "pending"
    tags: list[str] = attrs.Factory(list)

    # Field with alias (input/output name differs from attr name)
    email_addr: str = field(alias="emailAddress")

    # Field with separate input/output aliases
    k8s_ns: str = field(
        validation_alias="k8sNamespace",     # Accept this key on input
        serialisation_alias="namespace",      # Emit this key on output
    )

    # Field excluded from serialization
    internal_id: int = field(exclude=True)
```

### field() Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `alias` | `str` | Both input and output alias |
| `validation_alias` | `str` | Input-only alias (for .parse()) |
| `serialisation_alias` | `str` | Output-only alias (for unstructure) |
| `exclude` | `bool` | Exclude from serialization output |
| All `attrs.field()` params | — | `default`, `factory`, `validator`, `converter`, etc. |

## Parsing and Validation

```python
# Via classmethod (@inscribed required)
user = User.parse(data)                     # dict -> validated instance
user = User.parse(data, strict=True)        # Reject unexpected keys
user = User.parse(data, coerce=False)       # Strict types, no coercion
user = User.parse(data, converter=my_conv)  # Custom converter

# Via function (no @inscribed needed)
from rune import parse, validate
user = parse(data, User)

# Validate existing instance
validated = User.validate(user)
validated = validate(user)

# Normal attrs construction — NO validation, NO coercion
user = User(name="Alice", age=30, ...)
```

## Constraints

All constraints use `typing.Annotated`:

```python
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

## Validators

```python
import attrs
from rune import field_validator, model_validator

@attrs.define
class Plan:
    start_time: datetime
    end_time: datetime
    target_version: str | None = None

    @field_validator("target_version")
    @classmethod
    def validate_target(cls, v: str | None) -> str | None:
        if v is not None:
            Version(v)  # Raises ValueError if invalid
        return v

    @model_validator
    @classmethod
    def validate_dates(cls, instance):
        if instance.end_time <= instance.start_time:
            raise ValueError("end_time must be after start_time")
        return instance
```

## Converter

```python
from rune import Converter

c = Converter(
    mode="python",            # Default mode: "python" | "json"
    by_alias=False,           # Use aliases by default in unstructure
    type_registry=None,       # Custom TypeRegistry (uses default if None)
    forbid_extra_keys=False,  # Reject unknown keys during structure
)

# Structure: dict -> instance
user = c.structure(data, User)

# Unstructure: instance -> dict
d = c.unstructure(user)
d = c.unstructure(user, mode="json")           # JSON-safe types
d = c.unstructure(user, by_alias=True)         # Use aliases as keys
d = c.unstructure(user, mode="json", by_alias=True)  # Both
```

## TypeRegistry

```python
from rune import TypeRegistry, default_registry

# default_registry includes: datetime, date, time, UUID, Decimal, Path, IPv4, IPv6

registry = default_registry.copy()
registry.register(
    Version,
    structure_fn=lambda v, _: Version(v) if isinstance(v, str) else v,
    unstructure_fn=lambda v: str(v),
    schema_fn=lambda _: {"type": "string", "pattern": r"^\d+\.\d+\.\d+"},
    validate_fn=lambda v: None,
)

c = Converter(type_registry=registry)
```

## CoercionRegistry

```python
from rune import CoercionRegistry

# Default coercions: str->int/float/Decimal/bool/datetime/UUID, int->float, etc.
registry = CoercionRegistry.with_defaults()
registry.register(str, Version, lambda v: Version(v))

user = User.parse(data, coercion_registry=registry)
```

## Error Handling

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
```

Each error dict: `{"loc": ("name",), "msg": "...", "type": "min_length", "input": "", "ctx": {"min_length": 1}}`

**ErrorTypes constants:** `MISSING`, `UNEXPECTED`, `TYPE_ERROR`, `TYPE_ERROR_INT`, `GREATER_THAN`, `LESS_THAN`, `MIN_LENGTH`, `MAX_LENGTH`, `PATTERN`, `MIN_ITEMS`, `UNIQUE_ITEMS`, `VALUE_ERROR`

## Sentinels

```python
from rune import MISSING, UNSET

# MISSING — required value not provided (distinct from None)
# UNSET — value explicitly unset (for partial updates)
# Both are falsy but distinct from None and each other
```
