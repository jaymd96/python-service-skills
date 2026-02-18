# casbin

An **authorization library** that supports access control models including ACL, RBAC, and ABAC.

`casbin` (also known as PyCasbin) implements a flexible, policy-based authorization framework. You define an access control model (who can do what on which resource) in a declarative configuration file, and casbin's enforcer evaluates requests against your policies at runtime. The model and policy are cleanly separated, making it easy to switch authorization schemes without changing application code.

---

## Latest Stable Version

| Detail | Value |
|--------|-------|
| **Package** | `casbin` |
| **Latest version** | 1.36.3 (2024) |
| **Python support** | Python >= 3.6 |
| **License** | Apache 2.0 |
| **Repository** | https://github.com/casbin/pycasbin |
| **Documentation** | https://casbin.org/docs/overview |

The Python package is `casbin` on PyPI. The broader Casbin project spans many languages (Go, Java, Node.js, etc.) with compatible model/policy formats.

---

## Installation

```bash
pip install casbin
```

For database-backed policy storage, install an adapter:

```bash
pip install casbin-sqlalchemy-adapter   # SQLAlchemy (PostgreSQL, MySQL, SQLite)
pip install casbin-redis-adapter        # Redis
pip install casbin-mongodb-adapter      # MongoDB
```

---

## Core Concepts

### The Model

A casbin model defines four (or five) sections:

| Section | Key | Purpose |
|---------|-----|---------|
| **Request** | `[request_definition]` | Shape of an authorization request |
| **Policy** | `[policy_definition]` | Shape of policy rules |
| **Role** | `[role_definition]` | Role/group inheritance (RBAC) |
| **Matchers** | `[matchers]` | Expression that evaluates request against policy |
| **Effect** | `[policy_effect]` | How to combine multiple matching rules |

### Model File Example (ACL)

```ini
# model.conf
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = r.sub == p.sub && r.obj == p.obj && r.act == p.act
```

### Policy File Example

```csv
# policy.csv
p, alice, data1, read
p, bob, data2, write
p, alice, data2, read
```

---

## Core API Reference

### `Enforcer(model, policy)`

The central class. Loads a model and policy, then evaluces authorization requests.

```python
import casbin

# From files
e = casbin.Enforcer("path/to/model.conf", "path/to/policy.csv")

# From strings (inline model)
# Use casbin.Enforcer with a model object
```

### `enforce(sub, obj, act, ...)` -- Check Authorization

```python
e = casbin.Enforcer("model.conf", "policy.csv")

e.enforce("alice", "data1", "read")   # True
e.enforce("alice", "data1", "write")  # False
e.enforce("bob", "data2", "write")    # True
```

### Policy Management API

```python
# Get all policy rules
e.get_policy()
# [['alice', 'data1', 'read'], ['bob', 'data2', 'write'], ...]

# Add a policy rule
e.add_policy("eve", "data3", "read")

# Remove a policy rule
e.remove_policy("alice", "data1", "read")

# Check if a policy exists
e.has_policy("bob", "data2", "write")  # True

# Filter policies
e.get_filtered_policy(0, "alice")  # all rules where sub == "alice"
```

### RBAC (Role-Based Access Control)

#### Model for RBAC

```ini
# rbac_model.conf
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[role_definition]
g = _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act
```

#### Policy with Roles

```csv
# rbac_policy.csv
p, admin, data1, read
p, admin, data1, write
p, admin, data2, read
p, admin, data2, write
p, viewer, data1, read
p, viewer, data2, read

g, alice, admin
g, bob, viewer
```

#### Role Management API

```python
e = casbin.Enforcer("rbac_model.conf", "rbac_policy.csv")

# Check permissions (resolves roles)
e.enforce("alice", "data1", "write")  # True (alice is admin)
e.enforce("bob", "data1", "write")    # False (bob is viewer)

# Role management
e.add_role_for_user("charlie", "admin")
e.delete_role_for_user("charlie", "admin")
e.get_roles_for_user("alice")          # ['admin']
e.get_users_for_role("admin")          # ['alice']
e.has_role_for_user("alice", "admin")  # True

# Get implicit roles (including inherited)
e.get_implicit_roles_for_user("alice")

# Get all permissions for a user (including from roles)
e.get_implicit_permissions_for_user("alice")
# [['admin', 'data1', 'read'], ['admin', 'data1', 'write'], ...]
```

### RBAC with Domains (Multi-Tenancy)

```ini
# rbac_with_domains_model.conf
[request_definition]
r = sub, dom, obj, act

[policy_definition]
p = sub, dom, obj, act

[role_definition]
g = _, _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub, r.dom) && r.dom == p.dom && r.obj == p.obj && r.act == p.act
```

```csv
p, admin, tenant1, data1, read
p, admin, tenant1, data1, write

g, alice, admin, tenant1
g, alice, viewer, tenant2
```

```python
e.enforce("alice", "tenant1", "data1", "write")  # True
e.enforce("alice", "tenant2", "data1", "write")  # False
```

### ABAC (Attribute-Based Access Control)

ABAC uses object attributes in matchers instead of fixed strings:

```ini
# abac_model.conf
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub_rule, act

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = eval(p.sub_rule) && r.act == p.act
```

```python
import casbin

class User:
    def __init__(self, name, age):
        self.name = name
        self.age = age

e = casbin.Enforcer("abac_model.conf")
# Policies can reference r.sub attributes
e.add_policy("r.sub.age >= 18", "access")

user = User("alice", 25)
e.enforce(user, "/resource", "access")  # True
```

---

## Adapters

Adapters persist policies to external storage. Without an adapter, policies live only in memory and the CSV file.

### File Adapter (Default)

```python
e = casbin.Enforcer("model.conf", "policy.csv")
e.save_policy()  # write changes back to policy.csv
```

### SQLAlchemy Adapter

```python
import casbin
from casbin_sqlalchemy_adapter import Adapter

adapter = Adapter("sqlite:///casbin.db")
# or: Adapter("postgresql://user:pass@localhost/dbname")

e = casbin.Enforcer("model.conf", adapter)
e.load_policy()

# Changes are auto-persisted via the adapter
e.add_policy("alice", "data1", "read")
```

### Custom Adapter

Any adapter must implement the `persist.Adapter` interface:

```python
class MyAdapter:
    def load_policy(self, model):
        """Load all policy rules into the model."""
        ...

    def save_policy(self, model):
        """Save all policy rules from the model."""
        ...

    def add_policy(self, sec, ptype, rule):
        """Add a policy rule to storage."""
        ...

    def remove_policy(self, sec, ptype, rule):
        """Remove a policy rule from storage."""
        ...

    def remove_filtered_policy(self, sec, ptype, field_index, *field_values):
        """Remove filtered policy rules."""
        ...
```

---

## Watchers

Watchers enable distributed policy synchronization across multiple enforcer instances:

```python
from casbin import Enforcer
# from casbin_redis_watcher import RedisWatcher  # example

e = Enforcer("model.conf", adapter)

# watcher = RedisWatcher("redis://localhost:6379")
# e.set_watcher(watcher)

# When any instance modifies policy, other instances reload
```

---

## Middleware Integration

### Flask

```python
# pip install flask-authz
from flask import Flask
from flask_authz import CasbinEnforcer

app = Flask(__name__)
app.config["CASBIN_MODEL"] = "model.conf"
app.config["CASBIN_POLICY"] = "policy.csv"

casbin_enforcer = CasbinEnforcer(app)

@app.route("/data1")
@casbin_enforcer.enforcer
def data1():
    return "Authorized"
```

### Django

```python
# pip install django-casbin
# Add 'django_casbin' to INSTALLED_APPS
# Configure CASBIN_MODEL and CASBIN_ADAPTER in settings.py
```

### FastAPI

```python
# pip install fastapi-authz
from fastapi import FastAPI, Depends
from fastapi_authz import CasbinMiddleware
import casbin

app = FastAPI()

enforcer = casbin.Enforcer("model.conf", "policy.csv")
app.add_middleware(CasbinMiddleware, enforcer=enforcer)
```

---

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
