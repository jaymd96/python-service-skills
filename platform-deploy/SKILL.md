---
name: platform-deploy
description: Workflow for deploying and operating services on the Apollo CD platform running on Citadel K3s clusters. Covers registering products in Apollo catalog, creating releases, deploying entities, orchestration with constraint evaluation, spoke agent execution, monitoring on Citadel, and 48-hour node lifecycle. Use when deploying services to production or managing the platform. Triggers on deploy service, apollo deployment, platform operations, citadel cluster, node lifecycle, orchestration, release management, production deployment.
---

# Platform Deploy — Deploy & Operate on Apollo/Citadel

## Workflow Overview

```
1. Register product (Apollo catalog)
   → 2. Publish release (from Forge pipeline)
     → 3. Create entity (target environment)
       → 4. Orchestration engine evaluates constraints
         → 5. Approve plan (if required)
           → 6. Spoke agent executes (Helm install/upgrade)
             → 7. Monitor & operate (Citadel node lifecycle)
```

## Quick Deployment

```bash
# Register product
apollo product create --group com.example --name catalog-service

# Release was published by Forge pipeline (sls-publish)
# Verify it arrived:
apollo release list com.example:catalog-service

# Deploy to staging
apollo entity create \
  --product com.example:catalog-service \
  --version 1.0.0 \
  --environment staging

# Check status
apollo entity status ENTITY_ID

# Promote to production (orchestration evaluates constraints)
apollo entity create \
  --product com.example:catalog-service \
  --version 1.0.0 \
  --environment production
```

## Platform Architecture

```
Apollo Hub (control plane)
  ├── Product Catalog      — products, releases, channels
  ├── Orchestration Engine — constraint evaluation, plan creation
  ├── RBAC (Casbin)        — team-based access control
  └── Event System         — plan/entity lifecycle signals

Apollo Spoke Agent (per-cluster)
  ├── Polls Hub for plans
  ├── HelmChartOperator    — executes Helm install/upgrade/rollback
  └── Reports state back   — ExpectedStateK8s

Citadel (infrastructure)
  ├── K3s cluster          — Kubernetes runtime
  ├── Cilium CNI           — zero-trust networking
  ├── NodeManager          — cordon/drain lifecycle
  ├── HealthChecker        — 4 health states
  └── 48-hour node rotation — ephemeral security
```

## Key Concepts

### Entity states (6-state FSM)
```
UNMANAGED → PENDING → INSTALLING → RUNNING ↔ DEGRADED
                                       ↘ FAILED
```

### Constraint types
- **MaintenanceWindow** — cron-based time windows for changes
- **Dependency** — upstream services must be healthy
- **Suppression** — blocks all changes during configured periods

### Citadel node lifecycle
```
Create VM → Join K3s → Healthy (48h max) → Cordon → Drain → Destroy → Replace
```

## References

- **[deployment.md](references/deployment.md)** — Complete deployment workflow, orchestration, approval, rollback
- **[operations.md](references/operations.md)** — Monitoring, node lifecycle, network policies, troubleshooting, gotchas

## Grep Patterns

- `ProductCatalog|apollo product` — Find catalog operations
- `OrchestrationEngine|PlanType` — Find orchestration
- `SpokeAgent|apollo spoke` — Find agent operations
- `NodeManager|HealthChecker` — Find Citadel node lifecycle
- `EntityState|entity_state_changed` — Find entity lifecycle
