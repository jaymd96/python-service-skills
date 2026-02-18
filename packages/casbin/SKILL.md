---
name: casbin
description: Policy-based authorization supporting ACL, RBAC, and ABAC models. Use when implementing access control, role-based permissions, attribute-based authorization, or policy enforcement. Triggers on casbin, authorization, access control, RBAC, ABAC, ACL, policy enforcement, permissions.
---

# casbin — Authorization Framework (v1.36.3)

## Quick Start

```bash
pip install casbin
```

```python
import casbin

e = casbin.Enforcer("model.conf", "policy.csv")
e.enforce("alice", "data1", "read")  # True or False
```

## Key Patterns

### ACL model (simplest)
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

### RBAC — add role definition and use g() in matcher
```ini
[role_definition]
g = _, _
[matchers]
m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act
```

### Runtime policy management
```python
e.add_policy("bob", "data1", "read")
e.remove_policy("bob", "data1", "read")
e.add_grouping_policy("alice", "admin")  # assign role
e.enforce("alice", "data1", "write")     # True (via admin role)
```

## References

- **[api.md](references/api.md)** — Core API (Enforcer, enforce, policy management), model concepts, RBAC/ABAC, adapters, watchers, and middleware integration
- **[examples.md](references/examples.md)** — Gotchas (argument order, in-memory policies, role matchers) and complete examples (REST API authorization)
