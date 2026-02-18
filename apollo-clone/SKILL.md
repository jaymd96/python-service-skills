---
name: apollo-clone
description: Continuous delivery platform on k3s. Use when working with Product Catalog, Entity lifecycle (6-state FSM), OrchestrationEngine with constraint evaluation, SpokeAgent deployment, RBAC with Casbin, release management, Helm chart deployment, or the Apollo CLI. Triggers on apollo, CD platform, product catalog, entity management, orchestration engine, deployment, spoke agent, RBAC, release channel.
---

# Apollo-clone — CD Platform (v0.1.0)

## Quick Start

```bash
pip install apollo
```

```python
from apollo import Product, Release, Entity, ProductCatalog

catalog = ProductCatalog(":memory:")
product = Product(group_id="com.example", artifact_id="my-service", name="My Service")
catalog.create_product(product)
```

## Key Patterns

### Entity lifecycle (6-state FSM)
```
UNMANAGED -> PENDING -> INSTALLING -> RUNNING <-> DEGRADED
                                          \-> FAILED
```

### Module architecture
```python
# 11 core modules
apollo.models          # Data models with transitions FSM
apollo.catalog         # SQLModel persistence
apollo.orchestration   # Constraint evaluation engine
apollo.agent / spoke   # Deployment agent (Burr workflows)
apollo.auth            # RBAC with Casbin
apollo.events          # Blinker-based pub/sub
apollo.scheduling      # Cron maintenance windows
apollo.config          # Dynaconf configuration
apollo.services        # Version resolution, health
apollo.api             # FastAPI REST endpoints
apollo.cli             # Click + Rich CLI
```

## References

- **[api.md](references/api.md)** — Product, Release, Entity, ProductCatalog, OrchestrationEngine, SpokeAgent
- **[models.md](references/models.md)** — Data models, EntityState FSM, PlanType, constraints, RBAC
- **[orchestration.md](references/orchestration.md)** — Engine, Plans, constraints, CRD/Agent execution, scheduling
- **[examples.md](references/examples.md)** — Complete workflows, CLI reference, integration patterns, gotchas

## Grep Patterns

- `ProductCatalog|Product\(|Release\(` — Find catalog operations
- `EntityState|EntityHealth` — Find entity lifecycle
- `OrchestrationEngine|PlanType` — Find orchestration
- `SpokeAgent|apollo\.spoke` — Find agent framework
