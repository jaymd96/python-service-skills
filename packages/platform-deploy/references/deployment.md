# Platform Deploy — Deployment Workflow

> Part of the platform-deploy workflow. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [End-to-End Flow](#end-to-end-flow)
- [Product Registration](#product-registration)
- [Release Management](#release-management)
- [Entity Deployment](#entity-deployment)
- [Orchestration & Constraints](#orchestration--constraints)
- [Approval Workflows](#approval-workflows)
- [Spoke Agent Execution](#spoke-agent-execution)
- [Rollback](#rollback)
- [Promotion Strategy](#promotion-strategy)

## End-to-End Flow

```
Developer                    Forge Pipeline              Apollo Hub
   |                              |                          |
   |-- git push tag v1.0.0 ------>|                          |
   |                              |-- sls-package ---------> |
   |                              |-- sls-docker ----------> |
   |                              |-- sls-publish ---------> |
   |                              |                          |-- Create Release
   |                              |                          |
   |-- apollo entity create ----->|                          |
   |   (staging)                  |                          |-- Create Entity
   |                              |                          |-- Evaluate constraints
   |                              |                          |-- Create Plan
   |                              |                          |
   |                              |      Spoke Agent         |
   |                              |          |               |
   |                              |          |<-- Poll plans -|
   |                              |          |-- helm install |
   |                              |          |-- report state>|
   |                              |                          |
   |<---- entity RUNNING --------|                          |
   |                              |                          |
   |-- apollo entity create ----->|                          |
   |   (production)               |                          |-- Evaluate constraints
   |                              |                          |-- (maintenance window?)
   |                              |                          |-- (dependencies healthy?)
   |                              |                          |-- Create Plan (PENDING)
   |                              |                          |
   |-- apollo plan approve ------>|                          |-- Execute plan
```

## Product Registration

```python
from apollo import Product, ProductCatalog

catalog = ProductCatalog("apollo.db")

product = Product(
    group_id="com.example",
    artifact_id="catalog-service",
    name="Catalog Service",
)
catalog.create_product(product)
```

```bash
apollo product create --group com.example --name catalog-service
apollo product list
apollo product get com.example:catalog-service
```

## Release Management

Releases arrive via `sls-publish` (Forge pipeline) or can be created manually:

```python
from apollo.models import Release, ReleaseType

release = Release(
    product_id="com.example:catalog-service",
    version="1.0.0",
    release_type=ReleaseType.RELEASE,
    channel="stable",
)
catalog.create_release(release)
```

```bash
apollo release list com.example:catalog-service
apollo release create com.example:catalog-service --version 1.0.0 --channel stable

# Promote between channels
apollo release promote RELEASE_ID --channel stable

# Recall a bad release
apollo release deprecate RELEASE_ID --reason "CVE-2024-XXX"
```

### Release channels

| Channel | Purpose | Source |
|---------|---------|--------|
| `dev` | Development builds | `X.Y.Z-N-gHASH` versions |
| `beta` | Release candidates | `X.Y.Z-rcN` versions |
| `stable` | Production releases | `X.Y.Z` versions |

## Entity Deployment

An entity is a deployed instance of a product in a specific environment:

```python
from apollo.models import Entity, EntityState

entity = Entity(
    product_id="com.example:catalog-service",
    environment="staging",
    release_version="1.0.0",
)
catalog.create_entity(entity)
# State: UNMANAGED → PENDING (auto-transition on create)
```

```bash
apollo entity create \
  --product com.example:catalog-service \
  --version 1.0.0 \
  --environment staging

apollo entity list --environment staging
apollo entity status ENTITY_ID
```

### Entity state transitions

| From | To | Trigger |
|------|----|---------|
| `UNMANAGED` | `PENDING` | Entity registered |
| `PENDING` | `INSTALLING` | Plan execution begins |
| `INSTALLING` | `RUNNING` | Installation succeeds |
| `INSTALLING` | `FAILED` | Installation fails |
| `RUNNING` | `DEGRADED` | Health checks failing |
| `DEGRADED` | `RUNNING` | Health recovers |
| `RUNNING` | `FAILED` | Fatal error |

## Orchestration & Constraints

The `OrchestrationEngine` evaluates entities against constraints before creating plans:

```python
from apollo.orchestration import (
    OrchestrationEngine, ConstraintEvaluator,
    MaintenanceWindowConstraint,
    DependencyConstraint,
    SuppressionWindowConstraint,
)

engine = OrchestrationEngine(catalog=catalog, plan_storage=storage)

# Evaluate all entities — proposes plans for drift
plans = engine.evaluate_all()
```

### Constraint types

| Constraint | Blocks when |
|-----------|-------------|
| `MaintenanceWindowConstraint` | Outside configured cron window |
| `DependencyConstraint` | Upstream dependencies unhealthy or wrong version |
| `SuppressionWindowConstraint` | During configured suppression period (holiday freeze) |

### Constraint evaluation

All constraints are AND-ed. A single violation blocks the plan:

```python
evaluator = ConstraintEvaluator()
result = evaluator.evaluate(plan, [
    MaintenanceWindowConstraint(cron_expression="0 2 * * 6", duration_hours=4),
    DependencyConstraint(catalog=catalog),
    SuppressionWindowConstraint(start="2024-12-20", end="2025-01-02", reason="Holiday freeze"),
])

if result.satisfied:
    engine.execute_plan(plan.id)
else:
    print(f"Blocked: {result.violations}")
    # Plan state = BLOCKED with violation reasons
```

## Approval Workflows

Production deployments typically require approval:

```python
from apollo.orchestration import AccreditationRouter, ApprovalGate

router = AccreditationRouter(rules=[
    # Production upgrades require SRE approval
    ApprovalRule(environment="production", plan_types=[PlanType.UPGRADE],
                 required_roles=["sre"]),
    # Critical services need VP approval
    ApprovalRule(criticality=Criticality.CRITICAL,
                 required_roles=["vp-engineering"]),
])
```

```bash
# Approve a pending plan
apollo plan approve PLAN_ID --by ops@team.com

# List pending approvals
apollo plan list --state pending

# Cancel a plan
apollo plan cancel PLAN_ID --reason "no longer needed"
```

## Spoke Agent Execution

The `SpokeAgent` runs on each Citadel cluster and executes plans:

```python
from apollo.spoke.agent import SpokeAgent, SpokeAgentConfig

agent = SpokeAgent(SpokeAgentConfig(
    hub_url="https://hub.apollo.internal",
    namespace="default",
    poll_interval=30,
))
agent.start()
```

### Execution flow

```
SpokeAgent polls Hub
  → Receives approved plan
    → Dispatches to HelmChartOperator
      → helm install/upgrade/rollback
        → Reports ExpectedStateK8s back to Hub
          → Entity state updated (INSTALLING → RUNNING)
```

### Plan type → Helm operation

| PlanType | Helm command |
|----------|-------------|
| `install` | `helm install` |
| `upgrade` | `helm upgrade` |
| `rollback` | `helm rollback` |
| `uninstall` | `helm uninstall` |
| `config_update` | `helm upgrade` (values only) |

## Rollback

```bash
# Create rollback plan
apollo plan create --entity ENTITY_ID --type rollback --target-version 0.9.0

# Or automatic rollback on failure
# (if entity transitions INSTALLING → FAILED, orchestration engine proposes rollback)
```

```python
# Programmatic rollback
plan = Plan(
    entity_id=entity.id,
    plan_type=PlanType.ROLLBACK,
    target_version="0.9.0",
)
catalog.create_plan(plan)
engine.approve_plan(plan.id, approved_by="oncall@team.com")
engine.execute_plan(plan.id)
```

## Promotion Strategy

### Staged rollout

```bash
# 1. Deploy to staging
apollo entity create --product com.example:my-service --version 1.0.0 --environment staging

# 2. Wait for stability (monitor health, run integration tests)
apollo entity status STAGING_ENTITY_ID

# 3. Promote release to stable channel
apollo release promote RELEASE_ID --channel stable

# 4. Deploy to production (within maintenance window)
apollo entity create --product com.example:my-service --version 1.0.0 --environment production

# 5. Monitor
apollo entity list --environment production
```

### Canary deployment (with Transmute)

For zero-downtime migrations, combine Apollo entities with Transmute's dual-write proxy:

```
1. Deploy new version as canary entity (10% traffic)
2. Transmute migration runs (dual-write mode)
3. Monitor both entities
4. If healthy: promote canary to 100%, finalize migration
5. If unhealthy: rollback entity, rollback migration
```
