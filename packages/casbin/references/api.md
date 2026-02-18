# Casbin — API Reference

> Part of the casbin skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Core Concepts](#core-concepts)
  - [The Model](#the-model)
  - [Model File Example (ACL)](#model-file-example-acl)
  - [Policy File Example](#policy-file-example)
- [Core API Reference](#core-api-reference)
  - [Enforcer(model, policy)](#enforcermodel-policy)
  - [enforce(sub, obj, act, ...) -- Check Authorization](#enforcesub-obj-act----check-authorization)
  - [Policy Management API](#policy-management-api)
  - [RBAC (Role-Based Access Control)](#rbac-role-based-access-control)
  - [RBAC with Domains (Multi-Tenancy)](#rbac-with-domains-multi-tenancy)
  - [ABAC (Attribute-Based Access Control)](#abac-attribute-based-access-control)
- [Adapters](#adapters)
  - [File Adapter (Default)](#file-adapter-default)
  - [SQLAlchemy Adapter](#sqlalchemy-adapter)
  - [Custom Adapter](#custom-adapter)
- [Watchers](#watchers)
- [Middleware Integration](#middleware-integration)
  - [Flask](#flask)
  - [Django](#django)
  - [FastAPI](#fastapi)

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
