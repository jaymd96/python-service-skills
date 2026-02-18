# Apollo-Clone — Examples & Gotchas

> Part of the apollo-clone skill. See [SKILL.md](../SKILL.md) for overview.

## Product Lifecycle

```python
from apollo import Product, Release, Entity, ProductCatalog
from apollo.models import EntityState, ReleaseType, PlanType

# Initialize catalog
catalog = ProductCatalog("apollo.db")

# Register a product
product = Product(
    group_id="com.example",
    artifact_id="catalog-service",
    name="Catalog Service",
)
catalog.create_product(product)

# Create a release
release = Release(
    product_id="com.example:catalog-service",
    version="1.0.0",
    release_type=ReleaseType.RELEASE,
    channel="stable",
)
catalog.create_release(release)

# Deploy an entity
entity = Entity(
    product_id="com.example:catalog-service",
    environment="staging",
    release_version="1.0.0",
)
catalog.create_entity(entity)

# Transition entity state
catalog.transition_entity(entity.id, EntityState.RUNNING)
```

## Orchestration Workflow

```python
from apollo.orchestration import OrchestrationEngine, DatabasePlanStorage

engine = OrchestrationEngine(
    catalog=catalog,
    plan_storage=DatabasePlanStorage(catalog),
)

# Evaluate all entities — proposes plans for any drift
plans = engine.evaluate_all()
for plan in plans:
    print(f"{plan.entity_id}: {plan.plan_type} -> {plan.target_version}")

# Approve and execute
for plan in plans:
    engine.approve_plan(plan.id, approved_by="ops@team.com")
    result = engine.execute_plan(plan.id)
    print(f"Plan {plan.id}: {'SUCCESS' if result.success else 'FAILED'}")
```

## Constraint-Gated Deployment

```python
from apollo.orchestration import (
    ConstraintEvaluator,
    MaintenanceWindowConstraint,
    DependencyConstraint,
    SuppressionWindowConstraint,
)

evaluator = ConstraintEvaluator()
constraints = [
    MaintenanceWindowConstraint(cron_expression="0 2 * * 6", duration_hours=4),
    DependencyConstraint(catalog=catalog),
    SuppressionWindowConstraint(
        start="2024-12-20T00:00:00Z",
        end="2025-01-02T00:00:00Z",
        reason="Holiday freeze",
    ),
]

result = evaluator.evaluate(plan, constraints)
if result.satisfied:
    engine.execute_plan(plan.id)
else:
    print(f"Blocked: {result.violations}")
```

## Event Subscribers

```python
from apollo.events import (
    entity_state_changed, plan_failed, plan_completed, release_promoted,
)

@entity_state_changed.connect
def on_state_change(sender, entity_id, old_state, new_state, **kwargs):
    if new_state == "failed":
        pagerduty.alert(f"Entity {entity_id} entered FAILED state")

@plan_failed.connect
def on_plan_failed(sender, plan_id, error, **kwargs):
    slack.post(f"Plan {plan_id} failed: {error}")

@plan_completed.connect
def on_plan_completed(sender, plan_id, **kwargs):
    slack.post(f"Plan {plan_id} completed successfully")

@release_promoted.connect
def on_promotion(sender, release_id, channel, **kwargs):
    audit_log.record(f"Release {release_id} promoted to {channel}")
```

## Spoke Agent Setup

```python
from apollo.spoke.agent import SpokeAgent, SpokeAgentConfig

config = SpokeAgentConfig(
    hub_url="https://hub.apollo.internal",
    namespace="default",
    poll_interval=30,
    offline_queue_enabled=True,
)

agent = SpokeAgent(config)
agent.start()
# Agent polls hub, executes plans via HelmChartOperator,
# reports state via ExpectedStateK8s
```

## RBAC Setup

```python
from apollo.auth import Enforcer, Team, Role

enforcer = Enforcer()

# Create teams with roles
enforcer.add_team(Team(name="platform", roles=[Role.ADMIN]))
enforcer.add_team(Team(name="dev", roles=[Role.EDITOR]))

# Check permissions
can_deploy = enforcer.check("user@dev", "product:write", "com.example:my-service")
```

## CLI Reference

```bash
# Products
apollo product list
apollo product create --group com.example --name my-service
apollo product get com.example:my-service

# Releases
apollo release list com.example:my-service
apollo release create com.example:my-service --version 1.0.0 --channel stable
apollo release promote RELEASE_ID --channel stable
apollo release deprecate RELEASE_ID --reason "CVE-2024-XXX"

# Entities
apollo entity list --environment production
apollo entity status ENTITY_ID

# Plans
apollo plan list --state pending
apollo plan approve PLAN_ID --by ops@team.com
apollo plan cancel PLAN_ID --reason "no longer needed"

# Environments
apollo environment list
apollo environment create --name staging

# Maintenance
apollo maintenance create --cron "0 2 * * 6" --duration 4 --env production
apollo maintenance list

# Auth
apollo auth login
apollo auth whoami
apollo team list
apollo team create platform --role admin

# Spoke
apollo spoke status
apollo spoke agent start

# Output formats
apollo product list --format json
apollo product list --format yaml
apollo product list --format table  # default
```

## Gotchas

1. **Entity states are FSM-enforced**: You cannot transition from `UNMANAGED` directly to `RUNNING`. The `EntityStateMachine` validates all transitions and rejects invalid ones
2. **Plans require approval in production**: The `AccreditationRouter` enforces approval workflows. Bypass only with explicit override and audit trail
3. **Constraints are AND-ed**: All constraints must be satisfied for a plan to proceed. A single violation blocks execution
4. **Suppression windows are absolute**: Unlike maintenance windows (recurring cron), suppression windows are one-time date ranges that block all changes
5. **Release recall strategies matter**: `freeze` just prevents new installs, `warn` allows with notification, `force_rollback` actively rolls back running instances
6. **SpokeAgent needs network access**: The agent communicates with Hub via HTTP/WebSocket. Offline mode queues commands in Redis but requires eventual connectivity
7. **Casbin RBAC is hierarchical**: `admin` inherits all permissions from `editor`, `operator`, and `viewer`. Assign the minimum required role
8. **SQLModel persistence**: The `ProductCatalog` uses SQLModel with SQLite by default. Pass a connection string for PostgreSQL in production
9. **Events are synchronous by default**: Blinker signals fire synchronously. Long-running event handlers block the caller. Use background workers for expensive operations
10. **Version ordering is SLS-specific**: `ReleaseVersion` implements SLS ordering rules. `1.0.0 < 1.0.0-rc1` is false — release candidates sort before releases
