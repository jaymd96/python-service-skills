---
name: apollo-clone
description: Continuous delivery platform on k3s. Use when working with Product Catalog, Entity lifecycle (6-state FSM), OrchestrationEngine with constraint evaluation, SpokeAgent deployment, V3 auth (5-layer pipeline, Casbin RBAC, 18 RID types, operations registry), Compass resource namespace, release management, Helm chart deployment, or the Apollo CLI. Triggers on apollo, CD platform, product catalog, entity management, orchestration engine, deployment, spoke agent, RBAC, release channel, constraint evaluation, RID, compass, auth pipeline.
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
# 20 core modules
apollo.models          # Domain models with transitions FSM (76+ exports)
apollo.catalog         # SQLModel persistence + module registry + dependency graph
apollo.orchestration   # Constraint evaluation engine + analytics + overrides (40 exports)
apollo.spoke           # Deployment agents — Helm, node lifecycle, auth broker, state reporter
apollo.auth            # V3 auth — 5-layer pipeline, Casbin RBAC, 18 RID types, operations registry
apollo.rid             # Typed resource identifiers (18 RID types, ri.service.instance.type.locator)
apollo.compass         # Resource namespace — RID paths, project ownership, Compass hierarchy
apollo.events          # Blinker-based pub/sub (20+ signals)
apollo.services        # Version resolution, health, drift, rollback, config versioning (27 exports)
apollo.api             # FastAPI REST endpoints (50+ routes across 27 routers)
apollo.cli             # Click + Rich CLI with audit logging
apollo.config          # Dynaconf configuration management
apollo.scheduling      # Cron maintenance windows + suppression windows
apollo.db              # Alembic migrations (0001-0015)
apollo.spec            # Product specification — manifests, traits, secrets, substitution engine (63 exports)
apollo.helm            # Helm release management — client, values builder, async interface
apollo.client          # Hub API client — async HTTP with retry, pooling, auth
apollo.registry        # OCI artifact registry — ECR, GCR, ACR authenticators
apollo.webhook         # K8s mutating admission webhook — registry rewriting, pull secret injection
apollo.security        # Vulnerability scanning — Trivy, ClamAV, auto-recall engine
```

## References

- **[api.md](references/api.md)** — Product, Release, Entity, ProductCatalog, OrchestrationEngine, SpokeAgent, FastAPI routes
- **[models.md](references/models.md)** — Data models, EntityState FSM, PlanType, constraints, V3 auth system
- **[orchestration.md](references/orchestration.md)** — Engine, Plans, constraints, overrides, analytics, CRD/Agent execution
- **[examples.md](references/examples.md)** — Complete workflows, CLI reference, integration patterns, gotchas

## Grep Patterns

- `ProductCatalog|Product\(|Release\(` — Find catalog operations
- `EntityState|EntityHealth` — Find entity lifecycle
- `OrchestrationEngine|PlanType` — Find orchestration
- `SpokeAgent|apollo\.spoke` — Find agent framework
- `ApolloEnforcer|create_enforcer|require_operation` — Find V3 auth
- `AuthorizationPipeline|AuthorizationRequest` — Find auth pipeline
- `OperationRegistry|SEED_OPERATIONS` — Find operations registry
- `ResourceIdentifier|generate_.*_rid` — Find RID operations
- `CompassClient|CompassService|resolve_project` — Find Compass namespace
- `ConstraintResult|ConstraintType|ConstraintSeverity` — Find constraint system
- `QuorumConstraint|NetworkPolicyConstraint` — Find advanced constraints
- `OverrideService|EmergencyOverrideRequest` — Find override system
- `PlanAnalytics|PlanMetrics` — Find plan analytics
