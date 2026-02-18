# rule-engine -- Business Rule Evaluation for Python

## Overview

**rule-engine** is a lightweight Python library for defining and evaluating business rules as human-readable expressions. Rules are written as string expressions (similar to Python syntax) and can be evaluated against data objects (dictionaries, objects, etc.) to determine matches or filter collections. The library parses expressions into an AST that can be inspected, serialized, and safely evaluated without `eval()`.

**Key Characteristics:**

- **Version:** 4.5.0 (latest stable as of early 2026)
- **Python:** 3.8+
- **License:** BSD-3-Clause
- **Dependencies:** None (pure Python)
- **Thread Safety:** Yes (rules are immutable after creation)

**When to use rule-engine:**

- Evaluating user-defined business rules stored in a database or configuration
- Filtering datasets based on dynamic criteria
- Building rule-based access control or decision engines
- Allowing non-technical users to write conditions in a safe expression language
- Replacing scattered if/elif chains with configurable rules

**Core Design Principles:**

- **Safe evaluation** -- no arbitrary code execution, sandboxed expression language
- **Familiar syntax** -- expressions look like Python but are limited to safe operations
- **Type-aware** -- optional type checking catches expression errors before evaluation
- **AST-based** -- parsed rules can be introspected for display, optimization, or serialization

---

## Installation

```bash
pip install rule-engine
```

rule-engine is a pure Python package with no external dependencies.

---

## Core API

### `Rule(expression)`

The primary class. Create a rule from a string expression, then use it to evaluate data.

```python
import rule_engine

# Create a rule
rule = rule_engine.Rule("name == 'Alice' and age >= 21")

# Check if a single object matches
rule.matches({"name": "Alice", "age": 25})  # True
rule.matches({"name": "Bob", "age": 19})    # False
```

#### Constructor Parameters

```python
rule_engine.Rule(
    expression,       # str: the rule expression string
    context=None,     # Context: optional type/resolver context
)
```

### `rule.matches(thing)`

Returns `True` if the given object satisfies the rule expression, `False` otherwise.

```python
rule = rule_engine.Rule("price > 100 and category == 'electronics'")

product = {"name": "Laptop", "price": 999.99, "category": "electronics"}
rule.matches(product)  # True

product = {"name": "Book", "price": 15.99, "category": "books"}
rule.matches(product)  # False
```

The `thing` parameter can be a dictionary, or any object with attributes matching the names used in the expression (resolved via the context's resolver).

### `rule.filter(things)`

Returns a generator that yields only the items from `things` that match the rule.

```python
rule = rule_engine.Rule("age >= 18")

people = [
    {"name": "Alice", "age": 25},
    {"name": "Bob", "age": 17},
    {"name": "Charlie", "age": 30},
    {"name": "Diana", "age": 15},
]

adults = list(rule.filter(people))
# [{"name": "Alice", "age": 25}, {"name": "Charlie", "age": 30}]
```

`filter()` is lazy (returns a generator), so it works efficiently with large datasets.

### `rule.evaluate(thing)`

Returns the result of evaluating the expression against the given object. Unlike `matches()`, which always returns a boolean, `evaluate()` returns whatever the expression produces.

```python
rule = rule_engine.Rule("age * 2")
rule.evaluate({"age": 21})  # 42

rule = rule_engine.Rule("name")
rule.evaluate({"name": "Alice"})  # "Alice"
```

---

## Context

The `Context` class controls how symbols (variable names) in expressions are resolved and optionally provides type information for validation.

### `Context(resolver=None, type_resolver=None)`

```python
import rule_engine

# Default context: resolves symbols as dictionary keys
context = rule_engine.Context()

# Custom resolver: resolve symbols from object attributes
context = rule_engine.Context(
    resolver=rule_engine.resolve_attribute,
)

# With type information for validation
context = rule_engine.Context(
    type_resolver=rule_engine.type_resolver_from_dict({
        "name": rule_engine.DataType.STRING,
        "age": rule_engine.DataType.FLOAT,
        "active": rule_engine.DataType.BOOLEAN,
    }),
)
```

### Resolvers

A resolver is a callable `(thing, name) -> value` that extracts values from data objects.

```python
# Built-in resolvers:
rule_engine.resolve_attribute  # Uses getattr(thing, name)
# Default behavior: thing[name] (dictionary key access)

# Custom resolver:
def my_resolver(thing, name):
    """Case-insensitive dictionary lookup."""
    for key, value in thing.items():
        if key.lower() == name.lower():
            return value
    raise rule_engine.SymbolResolutionError(name)

context = rule_engine.Context(resolver=my_resolver)
rule = rule_engine.Rule("Name == 'Alice'", context=context)
rule.matches({"name": "Alice"})  # True (case-insensitive)
```

### Type Resolver

A type resolver enables compile-time type checking of expressions. If the expression references an unknown symbol or uses incompatible types, a `SymbolResolutionError` or `TypeError` is raised when the rule is created, not when it is evaluated.

```python
context = rule_engine.Context(
    type_resolver=rule_engine.type_resolver_from_dict({
        "name": rule_engine.DataType.STRING,
        "age": rule_engine.DataType.FLOAT,
        "tags": rule_engine.DataType.ARRAY(rule_engine.DataType.STRING),
    }),
)

# This works:
rule = rule_engine.Rule("name == 'Alice'", context=context)

# This raises at rule creation time (typo in symbol name):
rule = rule_engine.Rule("nme == 'Alice'", context=context)
# SymbolResolutionError: nme
```

---

## Expression Syntax Reference

### Supported Types

| Type | Examples | Notes |
|------|----------|-------|
| Boolean | `true`, `false` | Case-insensitive |
| Integer / Float | `42`, `3.14`, `-7` | Standard numeric literals |
| String | `'hello'`, `"world"` | Single or double quotes |
| Null | `null` | Represents None/null |
| Array | `[1, 2, 3]` | Ordered list literals |
| Set | `{1, 2, 3}` | Unordered, unique elements |
| Datetime | `d"2024-01-15"` | Date/datetime literals |
| Timedelta | `t"P1D"` | ISO 8601 duration |

### Comparison Operators

| Operator | Description |
|----------|-------------|
| `==` | Equal |
| `!=` | Not equal |
| `<` | Less than |
| `>` | Greater than |
| `<=` | Less than or equal |
| `>=` | Greater than or equal |

### Logical Operators

| Operator | Description |
|----------|-------------|
| `and` | Logical AND |
| `or` | Logical OR |
| `not` | Logical NOT |

### Arithmetic Operators

| Operator | Description |
|----------|-------------|
| `+` | Addition |
| `-` | Subtraction |
| `*` | Multiplication |
| `/` | Division (float) |
| `//` | Floor division |
| `%` | Modulo |
| `**` | Exponentiation |

### Membership and Identity

| Operator | Description |
|----------|-------------|
| `in` | Membership test |
| `not in` | Negative membership |
| `is null` | Null check |
| `is not null` | Non-null check |

### String Operations

| Operation | Description |
|-----------|-------------|
| `=~` or `=~~` | Regex match (`=~` case-sensitive, `=~~` case-insensitive) |
| `!~` or `!~~` | Regex non-match |
| `.length` | String length attribute |

### Collection Operations

| Operation | Description |
|-----------|-------------|
| `item in collection` | Membership test |
| `collection.length` | Number of elements |

### Attribute Access

Dotted attribute access is supported for nested data:

```python
rule = rule_engine.Rule("user.address.city == 'New York'")

data = {
    "user": {
        "address": {
            "city": "New York"
        }
    }
}
rule.matches(data)  # True
```

### Ternary Expression

```python
rule = rule_engine.Rule("age >= 18 ? 'adult' : 'minor'")
rule.evaluate({"age": 25})  # "adult"
rule.evaluate({"age": 15})  # "minor"
```

---

## AST Inspection

Rules are parsed into an Abstract Syntax Tree (AST) that can be programmatically inspected.

```python
rule = rule_engine.Rule("age >= 21 and name == 'Alice'")

# Access the AST
ast = rule.statement

# The AST is composed of Expression nodes
print(type(ast))  # LogicalExpression or similar

# Useful for debugging, displaying rules, or building rule editors
```

### Accessing Rule Text

```python
rule = rule_engine.Rule("age > 18")
print(rule.text)  # "age > 18"
```

---

## Complete Code Examples

### Dynamic Business Rules Engine

```python
import rule_engine

# Rules stored in a database or configuration
rules_config = [
    {
        "name": "Premium Discount",
        "condition": "customer_type == 'premium' and order_total >= 100",
        "discount": 0.15,
    },
    {
        "name": "Bulk Discount",
        "condition": "quantity >= 50",
        "discount": 0.10,
    },
    {
        "name": "New Customer",
        "condition": "is_new_customer == true and order_total >= 50",
        "discount": 0.05,
    },
]

def calculate_discount(order: dict) -> float:
    """Apply the first matching rule's discount to the order."""
    for rule_config in rules_config:
        rule = rule_engine.Rule(rule_config["condition"])
        if rule.matches(order):
            return rule_config["discount"]
    return 0.0

# Usage:
order = {
    "customer_type": "premium",
    "order_total": 250.00,
    "quantity": 10,
    "is_new_customer": False,
}
discount = calculate_discount(order)
print(f"Discount: {discount * 100}%")  # "Discount: 15.0%"
```

### Filtering with Type Safety

```python
import rule_engine

# Define the schema for type checking
context = rule_engine.Context(
    type_resolver=rule_engine.type_resolver_from_dict({
        "name": rule_engine.DataType.STRING,
        "age": rule_engine.DataType.FLOAT,
        "email": rule_engine.DataType.STRING,
        "active": rule_engine.DataType.BOOLEAN,
        "score": rule_engine.DataType.FLOAT,
    }),
)

# Rules with type checking -- errors caught at creation time
try:
    rule = rule_engine.Rule("age > 'twenty'", context=context)
except rule_engine.errors.EvaluationError as e:
    print(f"Type error caught: {e}")

# Valid rule
rule = rule_engine.Rule(
    "active == true and score >= 80 and age >= 18",
    context=context,
)

users = [
    {"name": "Alice", "age": 25, "email": "alice@example.com", "active": True, "score": 92},
    {"name": "Bob", "age": 17, "email": "bob@example.com", "active": True, "score": 88},
    {"name": "Charlie", "age": 30, "email": "charlie@example.com", "active": False, "score": 95},
    {"name": "Diana", "age": 22, "email": "diana@example.com", "active": True, "score": 75},
    {"name": "Eve", "age": 28, "email": "eve@example.com", "active": True, "score": 85},
]

eligible = list(rule.filter(users))
# [Alice (25, active, 92), Eve (28, active, 85)]
for user in eligible:
    print(f"  {user['name']}: score={user['score']}")
```

### Working with Object Attributes

```python
import rule_engine
from dataclasses import dataclass

@dataclass
class Employee:
    name: str
    department: str
    salary: float
    years_of_service: int

context = rule_engine.Context(resolver=rule_engine.resolve_attribute)

rule = rule_engine.Rule(
    "department == 'engineering' and years_of_service >= 5 and salary < 120000",
    context=context,
)

employees = [
    Employee("Alice", "engineering", 110000, 7),
    Employee("Bob", "engineering", 130000, 3),
    Employee("Charlie", "sales", 90000, 10),
    Employee("Diana", "engineering", 95000, 6),
]

for emp in rule.filter(employees):
    print(f"{emp.name} is eligible for a raise")
# Alice is eligible for a raise
# Diana is eligible for a raise
```

### Regex Matching

```python
import rule_engine

# Case-sensitive regex match
rule = rule_engine.Rule('email =~ ".*@company\\.com$"')
rule.matches({"email": "alice@company.com"})    # True
rule.matches({"email": "alice@gmail.com"})      # False

# Case-insensitive regex match
rule = rule_engine.Rule('name =~~ "^alice"')
rule.matches({"name": "Alice"})    # True
rule.matches({"name": "ALICE"})    # True
```

### Null Handling

```python
import rule_engine

rule = rule_engine.Rule("middle_name is not null")
rule.matches({"middle_name": "James"})  # True
rule.matches({"middle_name": None})     # False

# Combine with other conditions
rule = rule_engine.Rule("discount is not null and discount > 0")
rule.matches({"discount": 10})    # True
rule.matches({"discount": None})  # False
rule.matches({"discount": 0})     # False
```

### Date Comparisons

```python
import rule_engine
import datetime

rule = rule_engine.Rule('created_at >= d"2024-01-01" and created_at < d"2025-01-01"')

rule.matches({"created_at": datetime.date(2024, 6, 15)})   # True
rule.matches({"created_at": datetime.date(2023, 12, 31)})   # False
```

---

## Security Considerations

rule-engine is designed to be safe for evaluating user-provided expressions:

1. **No code execution** -- expressions cannot call functions, import modules, or execute arbitrary Python code.
2. **No attribute access to internals** -- you cannot access `__class__`, `__dict__`, or other dunder attributes.
3. **Limited operations** -- only comparison, arithmetic, logical, and membership operations are supported.
4. **No side effects** -- expressions are pure; they cannot modify the data being evaluated.

However, keep these precautions in mind:

- **Denial of service**: Extremely complex expressions or deeply nested logic could consume significant CPU time. Consider adding a timeout or complexity limit for user-supplied rules.
- **Data exposure**: The expression can access any key/attribute exposed by the resolver. Make sure the resolver only exposes data that the rule author is authorized to access.
- **Regex complexity**: The `=~` operator uses Python's `re` module. Malicious regex patterns (ReDoS) could cause exponential backtracking. Consider limiting regex use or adding timeout guards.

```python
# Limit exposed data by using a selective resolver
def safe_resolver(thing, name):
    allowed_fields = {"name", "age", "status", "category"}
    if name not in allowed_fields:
        raise rule_engine.SymbolResolutionError(name)
    return thing[name]

context = rule_engine.Context(resolver=safe_resolver)
```

---

## Gotchas and Common Mistakes

### 1. Numeric Types Are Floats by Default

rule-engine treats all numeric values as floats internally. Integer comparison works as expected, but be aware of floating-point precision issues.

```python
rule = rule_engine.Rule("price == 19.99")
# May have floating-point comparison issues for some values
```

### 2. Boolean Literals Are `true` / `false` (Lowercase)

Unlike Python's `True` / `False`, rule-engine uses lowercase `true` and `false`.

```python
# CORRECT:
rule = rule_engine.Rule("active == true")

# WRONG (these are treated as symbol references, not boolean literals):
# rule = rule_engine.Rule("active == True")
```

### 3. Strings Require Quotes in Expressions

Symbol names (variable references) are bare identifiers. String literals must be quoted.

```python
# 'alice' is a string literal:
rule = rule_engine.Rule("name == 'alice'")

# alice without quotes would be a symbol reference:
rule = rule_engine.Rule("name == alice")  # Compares name to the value of 'alice' field
```

### 4. Missing Keys Raise Errors

If the data object does not contain a key referenced in the expression, a `SymbolResolutionError` is raised at evaluation time.

```python
rule = rule_engine.Rule("age > 18")
rule.matches({"name": "Alice"})  # Raises SymbolResolutionError: age
```

Handle this by ensuring your data always includes the required fields, or by using a custom resolver with default values.

### 5. `filter()` Returns a Generator

`rule.filter()` returns a generator, not a list. If you need to iterate multiple times or get the length, convert it to a list first.

```python
results = rule.filter(items)       # Generator (one-time use)
results = list(rule.filter(items)) # List (reusable)
```

### 6. Regex Syntax Uses Python `re` Module

The `=~` operator uses Python's standard `re` module, not PCRE or another flavor. The pattern is matched against the entire string (uses `re.match` semantics on the full value).

### 7. Context Reuse

Create the `Context` once and reuse it across multiple rules for efficiency. Do not create a new context for every rule evaluation.

```python
# GOOD: reuse context
context = rule_engine.Context(type_resolver=...)
rule1 = rule_engine.Rule("age > 18", context=context)
rule2 = rule_engine.Rule("name == 'Alice'", context=context)

# WASTEFUL: creating context for each rule
rule1 = rule_engine.Rule("age > 18", context=rule_engine.Context(type_resolver=...))
```
