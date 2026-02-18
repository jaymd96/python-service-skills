# cattrs — Examples & Gotchas

> Part of the cattrs skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

1. [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
   - [Hook Registration Order Matters](#1-hook-registration-order-matters)
   - [Union Disambiguation](#2-union-disambiguation-cattrs-cannot-always-auto-detect)
   - [Missing Fields vs None Fields](#3-missing-fields-vs-none-fields)
   - [Converter Instance Sharing vs Isolation](#4-converter-instance-sharing-vs-isolation)
   - [Performance: GenConverter vs Converter](#5-performance-genconverter-vs-converter)
   - [Recursive / Self-Referential Types](#6-recursive--self-referential-types)
   - [Extra Keys in Input Dicts](#7-extra-keys-in-input-dicts)
   - [attrs Validators Run During Structuring](#8-attrs-validators-run-during-structuring)
   - [ClassValidationError for Detailed Errors](#9-classvalidationerror-for-detailed-errors)
2. [Complete Code Examples](#complete-code-examples)
   - [Example 1: Basic Structure and Unstructure](#example-1-basic-structure-and-unstructure)
   - [Example 2: Custom Hooks for Dates and Enums](#example-2-custom-hooks-for-dates-and-enums)
   - [Example 3: Tagged Unions for Polymorphism](#example-3-tagged-unions-for-polymorphism)
   - [Example 4: Nested attrs Classes with Lists and Optionals](#example-4-nested-attrs-classes-with-lists-and-optionals)
   - [Example 5: Preconf Converter for JSON APIs](#example-5-preconf-converter-for-json-apis)
   - [Example 6: GenConverter with Overrides](#example-6-genconverter-with-overrides-renameomit)
   - [Example 7: Full Pipeline -- API Request to Database and Back](#example-7-full-pipeline----api-request-to-database-and-back)
   - [Example 8: Structuring with include_subclasses](#example-8-structuring-with-include_subclasses)
   - [Example 9: Handling Complex Unions with Custom Disambiguation](#example-9-handling-complex-unions-with-custom-disambiguation)
   - [Example 10: Converter with Comprehensive Hook Setup](#example-10-converter-with-comprehensive-hook-setup)
3. [References (See Also)](#references)

---

## Gotchas and Common Mistakes

### 1. Hook Registration Order Matters

Predicate-based hooks (`register_structure_hook_func`) are checked in **reverse registration order** -- the last one registered is checked first. If two predicates both match a type, the one registered later wins.

```python
import cattrs

converter = cattrs.Converter()

# Registered first -- lower priority
converter.register_structure_hook_func(
    lambda t: hasattr(t, "__attrs_attrs__"),
    lambda v, t: f"attrs hook: {v}",
)

# Registered second -- higher priority (checked first)
converter.register_structure_hook_func(
    lambda t: hasattr(t, "__attrs_attrs__"),
    lambda v, t: f"override hook: {v}",
)

# The second hook wins because it was registered later
```

**Exact type hooks** (`register_structure_hook(type, func)`) always take precedence over predicate hooks, regardless of registration order.

**Best practice**: Register your most specific hooks last, or use exact type hooks when possible to avoid ambiguity.

### 2. Union Disambiguation (cattrs Cannot Always Auto-Detect)

cattrs **cannot** automatically disambiguate `Union[TypeA, TypeB]` when both types are attrs classes or dicts with overlapping structures. This is one of the most common sources of confusion.

```python
import attr
import cattrs
from typing import Union

@attr.s(auto_attribs=True)
class Cat:
    name: str
    indoor: bool

@attr.s(auto_attribs=True)
class Dog:
    name: str
    breed: str

# THIS WILL FAIL or produce incorrect results:
# cattrs.structure({"name": "Rex", "breed": "Lab"}, Union[Cat, Dog])
```

**Solutions:**

**Option A**: Use `configure_tagged_union` (recommended):

```python
from cattrs.strategies import configure_tagged_union

converter = cattrs.Converter()
configure_tagged_union(Union[Cat, Dog], converter, tag_name="_type")

data = {"_type": "Dog", "name": "Rex", "breed": "Lab"}
pet = converter.structure(data, Union[Cat, Dog])
assert isinstance(pet, Dog)
```

**Option B**: Register a custom union hook with disambiguation logic:

```python
converter = cattrs.Converter()

def structure_pet(data, _):
    if "breed" in data:
        return converter.structure(data, Dog)
    return converter.structure(data, Cat)

converter.register_structure_hook(Union[Cat, Dog], structure_pet)
```

**Option C**: Use `cattrs.disambiguators.create_default_dis_func` for automatic field-based disambiguation (works when types have unique required fields):

```python
# If Cat has 'indoor' and Dog has 'breed', cattrs can use those
# unique fields to tell them apart, but only if they are truly unique.
```

### 3. Missing Fields vs None Fields

cattrs distinguishes between a field being absent from the input dict and it being present with a `None` value.

```python
import attr
import cattrs

@attr.s(auto_attribs=True)
class User:
    name: str
    age: int = 0  # Has a default

# Missing field: uses the default
user = cattrs.structure({"name": "Alice"}, User)
assert user.age == 0  # Default applied

# Explicit None: raises (int is not Optional)
# cattrs.structure({"name": "Alice", "age": None}, User)
# ClassValidationError: expected int, got None
```

If a field should accept `None`, declare it as `Optional`:

```python
from typing import Optional

@attr.s(auto_attribs=True)
class User:
    name: str
    age: Optional[int] = None

# Now None is accepted
user = cattrs.structure({"name": "Alice", "age": None}, User)
assert user.age is None
```

### 4. Converter Instance Sharing vs Isolation

Do **not** share a single global `Converter` across unrelated subsystems if they need different serialization rules. Hooks registered on a converter affect all structuring/unstructuring done through that converter.

```python
import cattrs

# BAD: One converter with conflicting requirements
converter = cattrs.Converter()

# The API layer wants camelCase
# The database layer wants snake_case
# Both registering hooks on the same converter will conflict!

# GOOD: Separate converters
api_converter = cattrs.Converter()
db_converter = cattrs.Converter()
# Register API hooks on api_converter, DB hooks on db_converter
```

**Rule of thumb**: Create one converter per serialization boundary (API, database, cache, message queue, etc.).

### 5. Performance: GenConverter vs Converter

| Scenario | `Converter` | `GenConverter` |
|---|---|---|
| Simple attrs class (5 fields) | Baseline | ~2-3x faster |
| Deeply nested (3+ levels) | Baseline | ~3-5x faster |
| First-time structuring | Faster (no codegen) | Slower (generates code) |
| Repeated structuring | Baseline | Much faster |
| Field renaming/omitting | Not supported | Supported via `override()` |

**Guidance**: Use `GenConverter` in production for hot paths. Use plain `Converter` for scripts, tests, or when you need maximum flexibility. In modern cattrs versions, the base `Converter` also uses some code generation for attrs/dataclass types, so the gap is smaller than in older versions.

### 6. Recursive / Self-Referential Types

cattrs handles self-referential types, but you may need to be explicit about converter usage:

```python
from __future__ import annotations
import attr
import cattrs

@attr.s(auto_attribs=True)
class TreeNode:
    value: int
    children: list[TreeNode]

# This works out of the box
data = {"value": 1, "children": [{"value": 2, "children": []}]}
node = cattrs.structure(data, TreeNode)
```

However, if you use `GenConverter` with custom hooks and the type is self-referential, make sure you register the hook **before** any structuring call, as the generated code needs to reference the converter's hook for the recursive type.

### 7. Extra Keys in Input Dicts

By default, the base `Converter` ignores extra keys in the input dict. `GenConverter` also ignores them by default. If you want strict mode (reject unknown keys), you need to implement it yourself:

```python
import attr
import cattrs

@attr.s(auto_attribs=True)
class User:
    name: str
    age: int

# Extra key 'email' is silently ignored
user = cattrs.structure({"name": "Alice", "age": 30, "email": "alice@example.com"}, User)
assert user.name == "Alice"
# No 'email' attribute on User -- it was just dropped
```

### 8. attrs Validators Run During Structuring

If your attrs class has validators, they run when the class is instantiated during structuring. This can cause structuring to fail even when the data types are correct:

```python
import attr
import cattrs

@attr.s(auto_attribs=True)
class PositiveInt:
    value: int = attr.ib(validator=attr.validators.gt(0))

cattrs.structure({"value": 5}, PositiveInt)   # OK
cattrs.structure({"value": -1}, PositiveInt)  # Raises! Validator fails
```

### 9. `ClassValidationError` for Detailed Errors

cattrs raises `cattrs.errors.ClassValidationError` (a subclass of `BaseException`) that contains detailed per-field error information:

```python
import attr
import cattrs
from cattrs.errors import ClassValidationError

@attr.s(auto_attribs=True)
class User:
    name: str
    age: int

try:
    cattrs.structure({"name": "Alice", "age": "not_a_number"}, User)
except ClassValidationError as e:
    print(e.exceptions)  # List of per-field exceptions
    # Detailed error information about which field failed and why
```

---

## Complete Code Examples

### Example 1: Basic Structure and Unstructure

```python
"""
Basic structuring and unstructuring with attrs classes.
"""

import attr
import cattrs

@attr.s(auto_attribs=True)
class Address:
    street: str
    city: str
    zip_code: str

@attr.s(auto_attribs=True)
class User:
    name: str
    age: int
    address: Address

# Structure from a nested dict (e.g., JSON API response)
raw = {
    "name": "Alice",
    "age": 30,
    "address": {
        "street": "123 Main St",
        "city": "Springfield",
        "zip_code": "62701",
    },
}

user = cattrs.structure(raw, User)
assert isinstance(user, User)
assert isinstance(user.address, Address)
assert user.name == "Alice"
assert user.address.city == "Springfield"

# Unstructure back to a plain dict (e.g., for JSON serialization)
raw_back = cattrs.unstructure(user)
assert raw_back == raw
assert isinstance(raw_back, dict)
assert isinstance(raw_back["address"], dict)
```

### Example 2: Custom Hooks for Dates and Enums

```python
"""
Register custom hooks for datetime and Enum types.
"""

import attr
import cattrs
from datetime import datetime, date
from enum import Enum

class Priority(Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"

@attr.s(auto_attribs=True)
class Task:
    title: str
    priority: Priority
    due_date: date
    created_at: datetime

# Create a converter with custom hooks
converter = cattrs.Converter()

# Date: structured from "YYYY-MM-DD" strings
converter.register_structure_hook(date, lambda v, _: date.fromisoformat(v))
converter.register_unstructure_hook(date, lambda v: v.isoformat())

# Datetime: structured from ISO format strings
converter.register_structure_hook(datetime, lambda v, _: datetime.fromisoformat(v))
converter.register_unstructure_hook(datetime, lambda v: v.isoformat())

# Enums are handled automatically by cattrs, but you can customize:
# converter.register_structure_hook(Priority, lambda v, _: Priority(v))
# converter.register_unstructure_hook(Priority, lambda v: v.value)

# Structure from raw data
raw = {
    "title": "Write docs",
    "priority": "high",
    "due_date": "2024-03-01",
    "created_at": "2024-01-15T10:30:00",
}

task = converter.structure(raw, Task)
assert task.priority == Priority.HIGH
assert task.due_date == date(2024, 3, 1)
assert task.created_at == datetime(2024, 1, 15, 10, 30)

# Unstructure back
raw_back = converter.unstructure(task)
assert raw_back["due_date"] == "2024-03-01"
assert raw_back["created_at"] == "2024-01-15T10:30:00"
assert raw_back["priority"] == "high"
```

### Example 3: Tagged Unions for Polymorphism

```python
"""
Using configure_tagged_union for discriminated polymorphic deserialization.
"""

import attr
import cattrs
from typing import Union, List
from cattrs.strategies import configure_tagged_union

@attr.s(auto_attribs=True)
class TextBlock:
    content: str
    bold: bool = False

@attr.s(auto_attribs=True)
class ImageBlock:
    url: str
    alt_text: str = ""

@attr.s(auto_attribs=True)
class CodeBlock:
    code: str
    language: str = "python"

# The union of all block types
Block = Union[TextBlock, ImageBlock, CodeBlock]

@attr.s(auto_attribs=True)
class Document:
    title: str
    blocks: List[Block]

# Configure the converter with tagged union discrimination
converter = cattrs.Converter()
configure_tagged_union(
    Block,
    converter,
    tag_name="block_type",
    tag_generator=lambda t: t.__name__,  # Use class name as the tag
)

# Raw data with type tags
raw_doc = {
    "title": "My Document",
    "blocks": [
        {"block_type": "TextBlock", "content": "Hello, world!", "bold": True},
        {"block_type": "ImageBlock", "url": "https://example.com/img.png", "alt_text": "Example"},
        {"block_type": "CodeBlock", "code": "print('hi')", "language": "python"},
        {"block_type": "TextBlock", "content": "Goodbye!"},
    ],
}

doc = converter.structure(raw_doc, Document)
assert isinstance(doc.blocks[0], TextBlock)
assert isinstance(doc.blocks[1], ImageBlock)
assert isinstance(doc.blocks[2], CodeBlock)
assert isinstance(doc.blocks[3], TextBlock)
assert doc.blocks[0].bold is True
assert doc.blocks[1].url == "https://example.com/img.png"
assert doc.blocks[2].language == "python"
```

### Example 4: Nested attrs Classes with Lists and Optionals

```python
"""
Structuring deeply nested data with collections and optional fields.
"""

import attr
import cattrs
from typing import List, Optional

@attr.s(auto_attribs=True)
class Ingredient:
    name: str
    amount: str
    optional: bool = False

@attr.s(auto_attribs=True)
class Step:
    instruction: str
    duration_minutes: Optional[int] = None

@attr.s(auto_attribs=True)
class Recipe:
    name: str
    servings: int
    ingredients: List[Ingredient]
    steps: List[Step]
    notes: Optional[str] = None

raw = {
    "name": "Pancakes",
    "servings": 4,
    "ingredients": [
        {"name": "flour", "amount": "2 cups"},
        {"name": "milk", "amount": "1.5 cups"},
        {"name": "eggs", "amount": "2"},
        {"name": "blueberries", "amount": "1 cup", "optional": True},
    ],
    "steps": [
        {"instruction": "Mix dry ingredients", "duration_minutes": 5},
        {"instruction": "Add wet ingredients", "duration_minutes": 3},
        {"instruction": "Cook on griddle", "duration_minutes": 15},
        {"instruction": "Serve warm"},  # duration_minutes is None (missing)
    ],
    "notes": None,  # Explicitly null
}

recipe = cattrs.structure(raw, Recipe)

assert recipe.name == "Pancakes"
assert len(recipe.ingredients) == 4
assert recipe.ingredients[3].optional is True
assert recipe.steps[2].duration_minutes == 15
assert recipe.steps[3].duration_minutes is None
assert recipe.notes is None

# Round-trip
raw_back = cattrs.unstructure(recipe)
assert raw_back["ingredients"][0]["name"] == "flour"
```

### Example 5: Preconf Converter for JSON APIs

```python
"""
Using the preconf JSON converter for a real-world API client.
"""

import attr
import json
from datetime import datetime
from uuid import UUID
from typing import List, Optional
from cattrs.preconf.json import make_converter

# Create a JSON-optimized converter
converter = make_converter()

@attr.s(auto_attribs=True)
class Author:
    id: UUID
    name: str

@attr.s(auto_attribs=True)
class Comment:
    id: UUID
    author: Author
    body: str
    created_at: datetime

@attr.s(auto_attribs=True)
class BlogPost:
    id: UUID
    title: str
    body: str
    author: Author
    comments: List[Comment]
    published_at: Optional[datetime] = None

# Simulate an API response (already parsed from JSON)
api_response = {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "title": "Getting Started with cattrs",
    "body": "cattrs is a powerful serialization library...",
    "author": {
        "id": "11111111-2222-3333-4444-555555555555",
        "name": "Alice",
    },
    "comments": [
        {
            "id": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
            "author": {
                "id": "66666666-7777-8888-9999-000000000000",
                "name": "Bob",
            },
            "body": "Great article!",
            "created_at": "2024-01-15T14:30:00",
        },
    ],
    "published_at": "2024-01-15T10:00:00",
}

# Structure the API response into typed objects
post = converter.structure(api_response, BlogPost)

assert isinstance(post.id, UUID)
assert isinstance(post.author, Author)
assert isinstance(post.comments[0].created_at, datetime)
assert post.author.name == "Alice"
assert post.comments[0].body == "Great article!"

# Unstructure back to a JSON-compatible dict
raw = converter.unstructure(post)
json_str = json.dumps(raw, indent=2)

# The output is valid JSON with datetimes as ISO strings and UUIDs as strings
assert isinstance(raw["id"], str)
assert isinstance(raw["published_at"], str)
```

### Example 6: GenConverter with Overrides (Rename/Omit)

```python
"""
Using GenConverter with field renaming and omitting for API serialization.
"""

import attr
import cattrs
from cattrs.gen import make_dict_structure_fn, make_dict_unstructure_fn, override

@attr.s(auto_attribs=True)
class UserProfile:
    first_name: str
    last_name: str
    email_address: str
    password_hash: str = ""
    is_admin: bool = False
    login_count: int = 0

converter = cattrs.GenConverter()

# Configure unstructuring: Python snake_case -> JSON camelCase
# Also omit sensitive and internal fields
converter.register_unstructure_hook(
    UserProfile,
    make_dict_unstructure_fn(
        UserProfile,
        converter,
        first_name=override(rename="firstName"),
        last_name=override(rename="lastName"),
        email_address=override(rename="emailAddress"),
        password_hash=override(omit=True),           # Never expose passwords
        is_admin=override(rename="isAdmin"),
        login_count=override(rename="loginCount", omit_if_default=True),
    ),
)

# Configure structuring: JSON camelCase -> Python snake_case
converter.register_structure_hook(
    UserProfile,
    make_dict_structure_fn(
        UserProfile,
        converter,
        first_name=override(rename="firstName"),
        last_name=override(rename="lastName"),
        email_address=override(rename="emailAddress"),
        is_admin=override(rename="isAdmin"),
        login_count=override(rename="loginCount"),
    ),
)

# Unstructure: Python -> API
user = UserProfile(
    first_name="Alice",
    last_name="Smith",
    email_address="alice@example.com",
    password_hash="bcrypt$$2b$12$...",
    is_admin=False,
    login_count=0,
)

api_output = converter.unstructure(user)
assert api_output == {
    "firstName": "Alice",
    "lastName": "Smith",
    "emailAddress": "alice@example.com",
    "isAdmin": False,
    # password_hash is omitted
    # login_count is omitted (default value + omit_if_default)
}

# Structure: API -> Python
api_input = {
    "firstName": "Bob",
    "lastName": "Jones",
    "emailAddress": "bob@example.com",
    "isAdmin": True,
    "loginCount": 42,
}

user2 = converter.structure(api_input, UserProfile)
assert user2.first_name == "Bob"
assert user2.last_name == "Jones"
assert user2.is_admin is True
assert user2.login_count == 42
assert user2.password_hash == ""  # Default, since not in input
```

### Example 7: Full Pipeline -- API Request to Database and Back

```python
"""
Complete example: an API layer with its own converter, a storage layer with
its own converter, and data flowing between them.
"""

import attr
import cattrs
from cattrs.gen import make_dict_structure_fn, make_dict_unstructure_fn, override
from cattrs.preconf.json import make_converter as make_json_converter
from datetime import datetime, timezone
from uuid import UUID, uuid4
from typing import List, Optional

# --- Domain Model ---

@attr.s(auto_attribs=True)
class OrderItem:
    product_id: str
    quantity: int
    unit_price: float

@attr.s(auto_attribs=True)
class Order:
    id: UUID
    customer_email: str
    items: List[OrderItem]
    total: float
    created_at: datetime
    notes: Optional[str] = None

# --- API Converter (camelCase, JSON-compatible) ---

api_converter = make_json_converter()

# Register camelCase renaming for the API layer
api_converter.register_unstructure_hook(
    OrderItem,
    make_dict_unstructure_fn(
        OrderItem,
        api_converter,
        product_id=override(rename="productId"),
        unit_price=override(rename="unitPrice"),
    ),
)
api_converter.register_structure_hook(
    OrderItem,
    make_dict_structure_fn(
        OrderItem,
        api_converter,
        product_id=override(rename="productId"),
        unit_price=override(rename="unitPrice"),
    ),
)

api_converter.register_unstructure_hook(
    Order,
    make_dict_unstructure_fn(
        Order,
        api_converter,
        customer_email=override(rename="customerEmail"),
        created_at=override(rename="createdAt"),
    ),
)
api_converter.register_structure_hook(
    Order,
    make_dict_structure_fn(
        Order,
        api_converter,
        customer_email=override(rename="customerEmail"),
        created_at=override(rename="createdAt"),
    ),
)

# --- Storage Converter (snake_case, internal format) ---

db_converter = make_json_converter()
# Uses defaults -- snake_case matches the attrs field names

# --- Simulate the flow ---

# 1. API request comes in (camelCase JSON)
api_request = {
    "customerEmail": "alice@example.com",
    "items": [
        {"productId": "SKU-001", "quantity": 2, "unitPrice": 29.99},
        {"productId": "SKU-042", "quantity": 1, "unitPrice": 49.99},
    ],
}

# 2. Structure the API input (partial -- we add server-side fields)
items = api_converter.structure(api_request["items"], List[OrderItem])
total = sum(item.unit_price * item.quantity for item in items)

order = Order(
    id=uuid4(),
    customer_email=api_request["customerEmail"],
    items=items,
    total=total,
    created_at=datetime.now(timezone.utc),
)

# 3. Store in database format (snake_case)
db_record = db_converter.unstructure(order)
assert "customer_email" in db_record  # snake_case
assert "created_at" in db_record

# 4. Load from database
loaded_order = db_converter.structure(db_record, Order)
assert loaded_order.id == order.id

# 5. Send API response (camelCase)
api_response = api_converter.unstructure(loaded_order)
assert "customerEmail" in api_response  # camelCase
assert "createdAt" in api_response

# The API response is JSON-serializable
import json
json_str = json.dumps(api_response)
```

### Example 8: Structuring with `include_subclasses`

```python
"""
Polymorphic structuring using include_subclasses.
"""

import attr
import cattrs
from cattrs.strategies import include_subclasses
from typing import List

@attr.s(auto_attribs=True)
class Notification:
    recipient: str
    message: str

@attr.s(auto_attribs=True)
class EmailNotification(Notification):
    subject: str

@attr.s(auto_attribs=True)
class SMSNotification(Notification):
    phone_number: str

@attr.s(auto_attribs=True)
class PushNotification(Notification):
    device_token: str
    badge_count: int = 0

converter = cattrs.Converter()
include_subclasses(Notification, converter)

# cattrs tries each subclass and picks the one whose fields match
data = [
    {"recipient": "alice", "message": "Hello!", "subject": "Greetings"},
    {"recipient": "bob", "message": "Your code is ready", "phone_number": "+1234567890"},
    {"recipient": "carol", "message": "New update", "device_token": "abc123", "badge_count": 3},
]

notifications = [converter.structure(d, Notification) for d in data]

assert isinstance(notifications[0], EmailNotification)
assert isinstance(notifications[1], SMSNotification)
assert isinstance(notifications[2], PushNotification)
assert notifications[0].subject == "Greetings"
assert notifications[2].badge_count == 3
```

### Example 9: Handling Complex Unions with Custom Disambiguation

```python
"""
Custom disambiguation for Union types where automatic detection is not possible.
"""

import attr
import cattrs
from typing import Union, List

@attr.s(auto_attribs=True)
class SuccessResponse:
    data: dict
    status_code: int = 200

@attr.s(auto_attribs=True)
class ErrorResponse:
    error: str
    error_code: int
    status_code: int

@attr.s(auto_attribs=True)
class RedirectResponse:
    location: str
    status_code: int = 302

ApiResponse = Union[SuccessResponse, ErrorResponse, RedirectResponse]

converter = cattrs.Converter()

def structure_api_response(data, _):
    """Disambiguate based on which keys are present."""
    if "error" in data:
        return converter.structure(data, ErrorResponse)
    elif "location" in data:
        return converter.structure(data, RedirectResponse)
    else:
        return converter.structure(data, SuccessResponse)

converter.register_structure_hook(ApiResponse, structure_api_response)

# Test disambiguation
responses = [
    {"data": {"user": "alice"}, "status_code": 200},
    {"error": "Not found", "error_code": 404, "status_code": 404},
    {"location": "https://example.com/new", "status_code": 301},
]

results = [converter.structure(r, ApiResponse) for r in responses]
assert isinstance(results[0], SuccessResponse)
assert isinstance(results[1], ErrorResponse)
assert isinstance(results[2], RedirectResponse)
assert results[1].error == "Not found"
```

### Example 10: Converter with Comprehensive Hook Setup

```python
"""
A production-ready converter configuration with hooks for all common types.
"""

import attr
import cattrs
from datetime import datetime, date, time, timedelta
from decimal import Decimal
from enum import Enum
from pathlib import Path, PurePosixPath
from uuid import UUID
from typing import Any

def make_production_converter() -> cattrs.GenConverter:
    """Create a fully configured converter for production use."""
    converter = cattrs.GenConverter()

    # --- datetime family ---
    converter.register_structure_hook(
        datetime, lambda v, _: datetime.fromisoformat(v) if isinstance(v, str) else v
    )
    converter.register_unstructure_hook(datetime, lambda v: v.isoformat())

    converter.register_structure_hook(
        date, lambda v, _: date.fromisoformat(v) if isinstance(v, str) else v
    )
    converter.register_unstructure_hook(date, lambda v: v.isoformat())

    converter.register_structure_hook(
        time, lambda v, _: time.fromisoformat(v) if isinstance(v, str) else v
    )
    converter.register_unstructure_hook(time, lambda v: v.isoformat())

    converter.register_structure_hook(
        timedelta, lambda v, _: timedelta(seconds=v) if isinstance(v, (int, float)) else v
    )
    converter.register_unstructure_hook(timedelta, lambda v: v.total_seconds())

    # --- UUID ---
    converter.register_structure_hook(UUID, lambda v, _: UUID(v) if isinstance(v, str) else v)
    converter.register_unstructure_hook(UUID, lambda v: str(v))

    # --- Path ---
    converter.register_structure_hook(Path, lambda v, _: Path(v))
    converter.register_unstructure_hook(Path, lambda v: str(v))

    converter.register_structure_hook(PurePosixPath, lambda v, _: PurePosixPath(v))
    converter.register_unstructure_hook(PurePosixPath, lambda v: str(v))

    # --- Decimal ---
    converter.register_structure_hook(Decimal, lambda v, _: Decimal(str(v)))
    converter.register_unstructure_hook(Decimal, lambda v: str(v))

    # --- bytes ---
    import base64
    converter.register_structure_hook(
        bytes, lambda v, _: base64.b64decode(v) if isinstance(v, str) else v
    )
    converter.register_unstructure_hook(bytes, lambda v: base64.b64encode(v).decode("ascii"))

    return converter


# Usage
converter = make_production_converter()

@attr.s(auto_attribs=True)
class AuditEntry:
    id: UUID
    timestamp: datetime
    path: Path
    duration: timedelta
    payload_size: Decimal
    raw_data: bytes

entry = AuditEntry(
    id=UUID("12345678-1234-5678-1234-567812345678"),
    timestamp=datetime(2024, 1, 15, 10, 30, 0),
    path=Path("/var/log/app.log"),
    duration=timedelta(seconds=3.5),
    payload_size=Decimal("1024.50"),
    raw_data=b"hello world",
)

raw = converter.unstructure(entry)
assert raw == {
    "id": "12345678-1234-5678-1234-567812345678",
    "timestamp": "2024-01-15T10:30:00",
    "path": "/var/log/app.log",
    "duration": 3.5,
    "payload_size": "1024.50",
    "raw_data": "aGVsbG8gd29ybGQ=",
}

# Round-trip
entry2 = converter.structure(raw, AuditEntry)
assert entry2.id == entry.id
assert entry2.raw_data == b"hello world"
```

---

## References

- **GitHub**: https://github.com/python-attrs/cattrs
- **PyPI**: https://pypi.org/project/cattrs/
- **Documentation**: https://catt.rs/en/stable/
- **attrs documentation**: https://www.attrs.org/en/stable/ (cattrs' primary dependency)
- **Changelog**: https://catt.rs/en/stable/history.html
