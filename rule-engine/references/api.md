# rule-engine — API Reference

> Part of the rule-engine skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Core API](#core-api)
  - [Rule(expression)](#ruleexpression)
  - [rule.matches(thing)](#rulematchesthing)
  - [rule.filter(things)](#rulefilterthings)
  - [rule.evaluate(thing)](#ruleevaluatething)
- [Context](#context)
  - [Context(resolver, type\_resolver)](#contextresolvernone-type_resolvernone)
  - [Resolvers](#resolvers)
  - [Type Resolver](#type-resolver)
- [Expression Syntax Reference](#expression-syntax-reference)
  - [Supported Types](#supported-types)
  - [Comparison Operators](#comparison-operators)
  - [Logical Operators](#logical-operators)
  - [Arithmetic Operators](#arithmetic-operators)
  - [Membership and Identity](#membership-and-identity)
  - [String Operations](#string-operations)
  - [Collection Operations](#collection-operations)
  - [Attribute Access](#attribute-access)
  - [Ternary Expression](#ternary-expression)
- [AST Inspection](#ast-inspection)
- [Security Considerations](#security-considerations)

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
