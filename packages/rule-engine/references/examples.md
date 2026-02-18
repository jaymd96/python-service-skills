# rule-engine — Examples & Gotchas

> Part of the rule-engine skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Numeric Types Are Floats by Default](#1-numeric-types-are-floats-by-default)
  - [2. Boolean Literals Are true / false (Lowercase)](#2-boolean-literals-are-true--false-lowercase)
  - [3. Strings Require Quotes in Expressions](#3-strings-require-quotes-in-expressions)
  - [4. Missing Keys Raise Errors](#4-missing-keys-raise-errors)
  - [5. filter() Returns a Generator](#5-filter-returns-a-generator)
  - [6. Regex Syntax Uses Python re Module](#6-regex-syntax-uses-python-re-module)
  - [7. Context Reuse](#7-context-reuse)
- [Complete Code Examples](#complete-code-examples)
  - [Dynamic Business Rules Engine](#dynamic-business-rules-engine)
  - [Filtering with Type Safety](#filtering-with-type-safety)
  - [Working with Object Attributes](#working-with-object-attributes)
  - [Regex Matching](#regex-matching)
  - [Null Handling](#null-handling)
  - [Date Comparisons](#date-comparisons)

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
