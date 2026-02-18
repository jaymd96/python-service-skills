# attrs — Validators & Converters

> Part of the attrs skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Validators](#validators)
  - [Built-in Validators](#built-in-validators)
    - [instance_of(type)](#attrsvalidatorsinstance_oftype----type-checking)
    - [in_(options)](#attrsvalidatorsin_options----membership-check)
    - [matches_re(pattern)](#attrsvalidatorsmatches_repattern----regex-matching)
    - [lt(), gt(), le(), ge()](#attrsvalidatorslt-gt-le-ge----numeric-bounds)
    - [min_len(), max_len()](#attrsvalidatorsmin_len-max_len----length-validation)
    - [and_()](#attrsvalidatorsand_----composing-validators-all-must-pass)
    - [or_()](#attrsvalidatorsor_----composing-validators-any-must-pass)
    - [optional()](#attrsvalidatorsoptional----allow-none)
    - [deep_iterable()](#attrsvalidatorsdeep_iterable----validate-items-in-a-collection)
    - [deep_mapping()](#attrsvalidatorsdeep_mapping----validate-keys-and-values-in-a-dict)
  - [Custom Validators](#custom-validators)
    - [Method 1: @field.validator Decorator](#method-1-the-fieldvalidator-decorator-field-level)
    - [Method 2: Standalone Callable](#method-2-standalone-callable-validator)
    - [Method 3: Disabling Validators](#method-3-using-attrsvalidatorsset_disabled--attrsvalidatorsdisabled)
- [Converters](#converters)
  - [Converters with Type Information](#converters-with-type-information)
  - [Converters and Validators Interaction](#converters-and-validators-interaction)
  - [Optional Converter Pattern](#optional-converter-pattern)
  - [Pipe Converter](#pipe-converter)
  - [attrs.converters.to_bool()](#attrsconvertersto_bool)
- [Quick Reference — Validators](#quick-reference--validators)
- [Quick Reference — Converters](#quick-reference--converters)

## Validators

Validators run when an attribute is set (during `__init__` and, for mutable classes, on subsequent attribute assignment when `on_setattr` is configured). They receive three arguments: the instance, the attribute, and the value.

### Built-in Validators

#### `attrs.validators.instance_of(type)` -- Type Checking

```python
import attrs

@attrs.define
class User:
    name: str = attrs.field(validator=attrs.validators.instance_of(str))
    age: int = attrs.field(validator=attrs.validators.instance_of(int))

User("Alice", 30)     # OK
User("Alice", "old")  # raises TypeError: ("'age' must be <class 'int'>...")

# Multiple types:
@attrs.define
class Value:
    data: int | float = attrs.field(validator=attrs.validators.instance_of((int, float)))
```

#### `attrs.validators.in_(options)` -- Membership Check

```python
import attrs

@attrs.define
class Shirt:
    size: str = attrs.field(validator=attrs.validators.in_(["S", "M", "L", "XL"]))
    color: str = attrs.field(validator=attrs.validators.in_({"red", "blue", "green"}))

Shirt("M", "red")      # OK
Shirt("XXL", "red")    # raises ValueError: ("'size' must be in ['S', 'M', 'L', 'XL']...")
```

#### `attrs.validators.matches_re(pattern)` -- Regex Matching

```python
import attrs

@attrs.define
class Email:
    address: str = attrs.field(
        validator=attrs.validators.matches_re(r"^[^@]+@[^@]+\.[^@]+$")
    )

Email("user@example.com")  # OK
Email("invalid")            # raises ValueError
```

The `func` parameter controls which `re` method to use:

```python
# Use re.fullmatch (default is re.search):
attrs.validators.matches_re(r"^\d{5}$", func=re.fullmatch)
```

#### `attrs.validators.lt()`, `gt()`, `le()`, `ge()` -- Numeric Bounds

```python
import attrs

@attrs.define
class Temperature:
    celsius: float = attrs.field(validator=[
        attrs.validators.ge(-273.15),  # >= absolute zero
        attrs.validators.lt(1000),     # < 1000
    ])

@attrs.define
class Percentage:
    value: float = attrs.field(validator=[
        attrs.validators.ge(0),    # >= 0
        attrs.validators.le(100),  # <= 100
    ])
```

#### `attrs.validators.min_len()`, `max_len()` -- Length Validation

```python
import attrs

@attrs.define
class Password:
    value: str = attrs.field(validator=[
        attrs.validators.min_len(8),
        attrs.validators.max_len(128),
    ])

Password("short")              # raises ValueError (too short)
Password("validpassword123")   # OK
```

#### `attrs.validators.and_()` -- Composing Validators (All Must Pass)

Combines multiple validators; all must pass. Equivalent to passing a list, but more explicit.

```python
import attrs

positive_int = attrs.validators.and_(
    attrs.validators.instance_of(int),
    attrs.validators.gt(0),
)

@attrs.define
class Order:
    quantity: int = attrs.field(validator=positive_int)
```

**Note:** Passing a list of validators to `validator=[v1, v2, v3]` is implicitly `and_()`. The explicit form is useful when building reusable composite validators.

#### `attrs.validators.or_()` -- Composing Validators (Any Must Pass)

At least one validator must pass.

```python
import attrs

string_or_int = attrs.validators.or_(
    attrs.validators.instance_of(str),
    attrs.validators.instance_of(int),
)

@attrs.define
class Identifier:
    value: str | int = attrs.field(validator=string_or_int)

Identifier("abc")   # OK
Identifier(42)      # OK
Identifier(3.14)    # raises -- neither str nor int
```

#### `attrs.validators.optional()` -- Allow None

Wraps another validator to also accept `None`.

```python
import attrs

@attrs.define
class Profile:
    name: str = attrs.field(validator=attrs.validators.instance_of(str))
    bio: str | None = attrs.field(
        default=None,
        validator=attrs.validators.optional(attrs.validators.instance_of(str)),
    )

Profile("Alice")                # OK, bio=None
Profile("Alice", bio="Hello")   # OK
Profile("Alice", bio=42)        # raises TypeError
```

#### `attrs.validators.deep_iterable()` -- Validate Items in a Collection

Validates each element of an iterable, and optionally the iterable itself.

```python
import attrs

@attrs.define
class NumberList:
    values: list[int] = attrs.field(
        factory=list,
        validator=attrs.validators.deep_iterable(
            member_validator=attrs.validators.instance_of(int),
            iterable_validator=attrs.validators.instance_of(list),
        ),
    )

NumberList([1, 2, 3])      # OK
NumberList([1, "two", 3])  # raises TypeError on "two"
NumberList((1, 2, 3))      # raises TypeError -- not a list
```

#### `attrs.validators.deep_mapping()` -- Validate Keys and Values in a Dict

```python
import attrs

@attrs.define
class Headers:
    data: dict[str, str] = attrs.field(
        factory=dict,
        validator=attrs.validators.deep_mapping(
            key_validator=attrs.validators.instance_of(str),
            value_validator=attrs.validators.instance_of(str),
            mapping_validator=attrs.validators.instance_of(dict),
        ),
    )

Headers({"Content-Type": "text/html"})  # OK
Headers({1: "text/html"})               # raises TypeError on key
```

### Custom Validators

There are two ways to write custom validators.

#### Method 1: The `@field.validator` Decorator (Field-level)

Use this for validators tied to a specific field on a specific class. The decorated method receives the instance, attribute, and value.

```python
import attrs

@attrs.define
class Interval:
    lo: float
    hi: float

    @hi.validator
    def _check_hi(self, attribute, value):
        if value <= self.lo:
            raise ValueError(f"'hi' ({value}) must be greater than 'lo' ({self.lo})")

Interval(1.0, 5.0)   # OK
Interval(5.0, 1.0)   # raises ValueError
```

#### Method 2: Standalone Callable Validator

Any callable with the signature `(instance, attribute, value)` works as a validator. This is ideal for reusable validators shared across classes.

```python
import attrs

def validate_non_empty_string(instance, attribute, value):
    if not isinstance(value, str) or not value.strip():
        raise ValueError(f"'{attribute.name}' must be a non-empty string, got {value!r}")

@attrs.define
class Document:
    title: str = attrs.field(validator=validate_non_empty_string)
    body: str = attrs.field(validator=validate_non_empty_string)

Document("Hello", "World")  # OK
Document("", "World")       # raises ValueError
```

#### Method 3: Using `@attrs.validators.set_disabled()` / `@attrs.validators.disabled()`

You can temporarily disable all validators globally, which is useful for deserialization or testing:

```python
import attrs

with attrs.validators.disabled():
    # Validators do not run inside this block:
    item = Product(name="", price=-5)  # would normally fail validation
```

## Converters

Converters automatically transform input values during `__init__` (and on attribute assignment when `on_setattr` includes `convert`). They receive the value and return the converted value.

```python
import attrs

@attrs.define
class Config:
    port: int = attrs.field(converter=int)          # str -> int
    debug: bool = attrs.field(converter=bool)        # truthy/falsy -> bool
    tags: tuple = attrs.field(converter=tuple)        # any iterable -> tuple
    host: str = attrs.field(converter=str.lower)      # normalize to lowercase

c = Config(port="8080", debug=1, tags=["a", "b"], host="EXAMPLE.COM")
# Config(port=8080, debug=True, tags=('a', 'b'), host='example.com')
```

### Converters with Type Information

Since attrs 22.2.0, converters can optionally accept the value and the attribute:

```python
import attrs

def coerce_to_type(value, attribute=None):
    """Convert value to the field's annotated type."""
    if attribute and attribute.type:
        return attribute.type(value)
    return value
```

### Converters and Validators Interaction

Converters run **before** validators. This means validators see the already-converted value:

```python
import attrs

@attrs.define
class Port:
    number: int = attrs.field(
        converter=int,                          # "8080" -> 8080
        validator=attrs.validators.and_(
            attrs.validators.instance_of(int),  # sees 8080 (int), not "8080" (str)
            attrs.validators.ge(1),
            attrs.validators.le(65535),
        ),
    )

Port("8080")   # OK -- converter runs first, then validator sees 8080
Port("0")      # raises ValueError from ge(1) validator
```

### Optional Converter Pattern

A common pattern for fields that accept `None`:

```python
import attrs

def optional_int(value):
    """Convert to int, but pass through None."""
    if value is None:
        return None
    return int(value)

@attrs.define
class Query:
    limit: int | None = attrs.field(default=None, converter=optional_int)

Query(limit="50")   # Query(limit=50)
Query(limit=None)   # Query(limit=None)
Query()             # Query(limit=None)
```

Alternatively, use `attrs.converters.optional()` (available since attrs 17.1.0):

```python
import attrs

@attrs.define
class Query:
    limit: int | None = attrs.field(
        default=None,
        converter=attrs.converters.optional(int),
    )
```

### Pipe Converter

Chain multiple converters with `attrs.converters.pipe()`:

```python
import attrs

@attrs.define
class Tag:
    value: str = attrs.field(
        converter=attrs.converters.pipe(str, str.strip, str.lower)
    )

Tag("  HELLO  ")  # Tag(value='hello')
```

### `attrs.converters.to_bool()`

Converts common truthy/falsy string representations to `bool`:

```python
import attrs

@attrs.define
class Feature:
    enabled: bool = attrs.field(converter=attrs.converters.to_bool)

Feature("yes")   # Feature(enabled=True)
Feature("no")    # Feature(enabled=False)
Feature("true")  # Feature(enabled=True)
Feature("0")     # Feature(enabled=False)
Feature(1)       # Feature(enabled=True)
```

## Quick Reference -- Validators

| Validator | Description |
|---|---|
| `instance_of(type)` | Check `isinstance` |
| `in_(options)` | Check membership in a collection |
| `matches_re(pattern)` | Regex match |
| `gt(val)` / `ge(val)` | Greater than / greater-or-equal |
| `lt(val)` / `le(val)` | Less than / less-or-equal |
| `min_len(n)` / `max_len(n)` | Minimum / maximum length |
| `and_(*validators)` | All must pass |
| `or_(*validators)` | At least one must pass |
| `optional(validator)` | Allow `None`, otherwise run validator |
| `deep_iterable(member, iterable)` | Validate iterable elements |
| `deep_mapping(key, value, mapping)` | Validate mapping keys and values |

## Quick Reference -- Converters

| Converter | Description |
|---|---|
| `attrs.converters.optional(converter)` | Pass through `None`, otherwise run converter |
| `attrs.converters.pipe(*converters)` | Chain multiple converters |
| `attrs.converters.to_bool` | Convert strings/ints to `bool` |
