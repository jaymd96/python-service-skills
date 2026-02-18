---
name: rule-engine
description: Safe business rule evaluation with Python-like expression syntax. Use when evaluating user-defined rules, filtering data with dynamic criteria, building decision engines, or replacing scattered if/elif chains with configurable expressions. Triggers on rule engine, business rules, rule evaluation, expression engine, dynamic filtering, rule-engine.
---

# rule-engine — Business Rule Evaluation (v4.5.0)

## Quick Start

```bash
pip install rule-engine
```

```python
import rule_engine

rule = rule_engine.Rule("name == 'Alice' and age >= 21")
rule.matches({"name": "Alice", "age": 25})  # True
```

## Key Patterns

### Match and filter
```python
rule = rule_engine.Rule("price > 100 and category == 'electronics'")
rule.matches({"price": 999, "category": "electronics"})  # True
adults = list(rule.filter(people))  # generator of matching items
```

### Expression syntax
```python
# Comparison: ==, !=, <, >, <=, >=  |  Logical: and, or, not
# Containment: in, not in  |  Regex: =~  |  Null: value == null
rule_engine.Rule("status in ('active', 'pending')")
rule_engine.Rule("email =~ '.*@company\\.com'")
```

### Type-safe context
```python
context = rule_engine.Context(
    type_resolver=rule_engine.type_resolver_from_dict({
        "name": rule_engine.DataType.STRING,
        "age": rule_engine.DataType.FLOAT,
    })
)
rule = rule_engine.Rule("age > 18", context=context)
```

## References

- **[api.md](references/api.md)** — Core API (Rule, Context, resolvers), expression syntax, AST inspection, and security considerations
- **[examples.md](references/examples.md)** — Gotchas (floats, booleans, missing keys) and complete examples (business rules, filtering, regex, dates)
