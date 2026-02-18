# Casbin — Examples & Gotchas

> Part of the casbin skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Model and Policy Are Separate Concerns](#1-model-and-policy-are-separate-concerns)
  - [2. enforce() Argument Order Must Match request\_definition](#2-enforce-argument-order-must-match-request_definition)
  - [3. Policy Changes Are In-Memory by Default](#3-policy-changes-are-in-memory-by-default)
  - [4. RBAC Requires g() in Matchers](#4-rbac-requires-g-in-matchers)
  - [5. The Default Effect Is "Allow-Override"](#5-the-default-effect-is-allow-override)
  - [6. Loading Model from String](#6-loading-model-from-string)
- [Complete Examples](#complete-examples)
  - [Example: REST API Authorization](#example-rest-api-authorization)
- [See Also](#see-also)

## Gotchas and Common Mistakes

### 1. Model and Policy Are Separate Concerns

The model defines the *structure* of your access control (what fields exist, how they match). The policy defines the actual *rules*. Changing from ACL to RBAC means changing the model file, not the application code.

### 2. `enforce()` Argument Order Must Match `request_definition`

If your model defines `r = sub, obj, act`, you must call `e.enforce(sub, obj, act)` in that exact order.

### 3. Policy Changes Are In-Memory by Default

After `add_policy()` or `remove_policy()`, call `e.save_policy()` to persist to the file adapter. Database adapters typically auto-persist.

### 4. RBAC Requires `g()` in Matchers

For role inheritance to work, the matcher must use `g(r.sub, p.sub)` rather than `r.sub == p.sub`.

### 5. The Default Effect Is "Allow-Override"

`e = some(where (p.eft == allow))` means: if any matching policy says allow, the result is allow. For deny-override, use:

```ini
e = !some(where (p.eft == deny))
```

### 6. Loading Model from String

```python
import casbin
from casbin import model

m = model.Model()
m.load_model_from_text("""
[request_definition]
r = sub, obj, act
...
""")
e = casbin.Enforcer(m, "policy.csv")
```

---

## Complete Examples

### Example: REST API Authorization

```python
import casbin

# Model: role-based with URL path matching
model_text = """
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[role_definition]
g = _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub) && keyMatch(r.obj, p.obj) && regexMatch(r.act, p.act)
"""

# keyMatch: /api/users/123 matches /api/users/*
# regexMatch: GET matches GET

e = casbin.Enforcer()
m = casbin.model.Model()
m.load_model_from_text(model_text)
e.model = m

# In practice, load from a file or database adapter
e.add_policy("admin", "/api/*", ".*")
e.add_policy("user", "/api/users/:id", "GET")
e.add_grouping_policy("alice", "admin")
e.add_grouping_policy("bob", "user")

print(e.enforce("alice", "/api/users/1", "DELETE"))  # True (admin)
print(e.enforce("bob", "/api/users/1", "GET"))       # True (user)
print(e.enforce("bob", "/api/users/1", "DELETE"))     # False (user)
```

---

## See Also

- [Casbin Official Documentation](https://casbin.org/docs/overview)
- [PyCasbin on GitHub](https://github.com/casbin/pycasbin)
- [Casbin Online Editor](https://casbin.org/editor/) -- test models and policies in the browser
- [Adapter List](https://casbin.org/docs/adapters)
