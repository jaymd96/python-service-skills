# Rune — Serialization & Schema

> Part of the rune skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [Converter Mode Behavior](#converter-mode-behavior)
- [Aliases](#aliases)
- [Computed Fields](#computed-fields)
- [Discriminated Unions](#discriminated-unions)
- [JSON Schema Generation](#json-schema-generation)

## Converter Mode Behavior

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

## Aliases

### Three-tier alias system

```python
from rune import field

# Single alias (used for both input and output)
email: str = field(alias="emailAddress")

# Separate input/output aliases
k8s_ns: str = field(
    validation_alias="k8sNamespace",     # Accept this key on input (.parse())
    serialisation_alias="namespace",      # Emit this key on output (unstructure)
)
```

### Automatic camelCase aliases

```python
from rune import camel_case_aliases, inscribed
import attrs

@inscribed
@camel_case_aliases
@attrs.define
class User:
    first_name: str     # alias: "firstName"
    last_name: str      # alias: "lastName"
    email_address: str  # alias: "emailAddress"

user = User.parse({"firstName": "Alice", "lastName": "Smith", "emailAddress": "a@b.com"})
```

### Case conversion helpers

```python
from rune import to_camel, to_pascal, to_kebab, to_snake

to_camel("user_name")     # "userName"
to_pascal("user_name")    # "UserName"
to_kebab("user_name")     # "user-name"
to_snake("userName")      # "user_name"
```

## Computed Fields

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

## Discriminated Unions

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

payment = c.structure({"type": "credit_card", "number": "4111..."}, PaymentMethod)
assert isinstance(payment, CreditCard)
```

## JSON Schema Generation

```python
from rune import json_schema

schema = json_schema(User)
schema = json_schema(User, by_alias=True)              # Aliases as property names
schema = json_schema(User, mode="serialisation")        # Includes computed fields
schema = json_schema(User, title="UserModel", description="A user record")
```

### Type to JSON Schema mapping

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
