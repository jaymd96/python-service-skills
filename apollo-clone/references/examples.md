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
from datetime import datetime, timedelta, UTC
from apollo.orchestration import (
    ConstraintEvaluator,
    MaintenanceWindow, MaintenanceWindowConstraint,
    DependencyConstraint, DependencyRequirement, ServiceHealth,
    SuppressionWindowConstraint, SuppressionWindow, SuppressionType,
)

# Maintenance windows use structured MaintenanceWindow objects (not cron strings)
window = MaintenanceWindow(
    name="Saturday maintenance",
    start_time=datetime(2024, 12, 7, 2, 0, tzinfo=UTC),
    end_time=datetime(2024, 12, 7, 6, 0, tzinfo=UTC),
    allows_downtime=False,
)

maintenance = MaintenanceWindowConstraint(
    name="Production Maintenance",
    windows=[window],
)

# Dependency constraints use DependencyRequirement objects
dep_req = DependencyRequirement(
    service_name="auth-service",
    min_version="2.0.0",
    required_health=ServiceHealth.HEALTHY,
    is_critical=True,
)
dependency = DependencyConstraint(
    name="Service Dependencies",
    requirements=[dep_req],
)

# Suppression windows use SuppressionWindow objects (not raw start/end strings)
suppression_window = SuppressionWindow(
    name="Holiday freeze",
    start_time=datetime(2024, 12, 20, 0, 0, tzinfo=UTC),
    end_time=datetime(2025, 1, 2, 0, 0, tzinfo=UTC),
    suppression_type=SuppressionType.ALL,
    reason="Holiday freeze",
    allows_rollback=True,
)
suppression = SuppressionWindowConstraint(
    name="Holiday Suppression",
    windows=[suppression_window],
)

# Evaluate constraints against a context dict (not a Plan object)
evaluator = ConstraintEvaluator()
context = {"target_service": "catalog-service", "is_rollback": False}
result = evaluator.evaluate(context, [maintenance, dependency, suppression])

if result.satisfied:
    engine.execute_plan(plan.id)
else:
    print(f"Blocked: {result.message}")
    print(f"  Constraint: {result.constraint_type.value}")
    print(f"  Severity: {result.severity.value}")
    if result.reevaluate_at:
        print(f"  Retry after: {result.reevaluate_at}")
```

## V3 Auth Setup

```python
from apollo.auth import (
    ApolloEnforcer, create_enforcer, require_operation,
    OperationRegistry, SEED_OPERATIONS,
    AuthorizationPipeline, AuthorizationRequest,
    create_project_with_admin, add_project_member,
)

# Create enforcer (project-scoped, not team-scoped)
enforcer = create_enforcer(database_url="sqlite:///apollo.db")

# Seed the operations registry (70+ predefined operations)
registry = OperationRegistry(session=db_session)
registry.seed(SEED_OPERATIONS)

# Set up project with admin
create_project_with_admin(project_id="project-456", admin_user_id="ri.auth.main.user.1234")
add_project_member(project_id="project-456", user_id="ri.auth.main.user.5678", role="editor")

# Check permissions (6-tuple model: user, org, project, resource_type, resource_id, action)
can_deploy = enforcer.check_permission(
    user_id="ri.auth.main.user.1234",
    org_rid="ri.auth.main.org.default",
    project_id="project-456",
    resource_type="product",
    resource_id="com.example:my-service",
    action="write",
)

# Decorator usage — operations follow service:action-resource convention
@require_operation("catalog:read-product")
async def list_products(request): ...

@require_operation("orchestration:execute-plan")
async def execute_plan(request, plan_id: str): ...

# Full authorization pipeline (5 layers: Cross-Org -> RBAC -> MAC -> MFA -> ABAC)
pipeline = AuthorizationPipeline(enforcer=enforcer, enable_mfa_rules=True)
result = pipeline.evaluate(AuthorizationRequest(
    user_id="ri.auth.main.user.1234",
    org_rid="ri.auth.main.org.default",
    project_id="project-456",
    resource_type="product",
    resource_id="com.example:my-service",
    action="write",
    mfa_verified=True,
))
# result.allowed, result.denied_by, result.requires_mfa
```

## Compass Resource Namespace

```python
from apollo.compass import CompassService, CompassMode, create_compass_service
from apollo.rid import generate_product_rid, ResourceIdentifier

compass = create_compass_service(session=db_session, mode=CompassMode.LOCAL)

# Build hierarchy: Org -> Space -> Project -> Resource
space = compass.create_space(org_rid, name="Platform", slug="platform")
project = compass.create_project(space["rid"], name="Catalog", slug="catalog")
product_rid = generate_product_rid()
compass.register_resource(project["rid"], str(product_rid), "My Product", "product")

# O(1) project resolution (critical for V4 auth with @require_resource_operation)
resolved = compass.resolve_project(str(product_rid))
# resolved["project_rid"] -> project's RID for auth scoping
```

## Event Subscribers

```python
from apollo.events import (
    entity_state_changed, plan_failed, plan_completed, release_promoted,
    override_approved, init_handlers,
)

# Initialize built-in handlers (notifications, audit, metrics)
init_handlers()

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

@override_approved.connect
def on_override(sender, override_id, approved_by, **kwargs):
    audit_log.record(f"Override {override_id} approved by {approved_by}")
```

## Override Workflow

```python
from datetime import timedelta
from apollo.orchestration import (
    EmergencyOverrideRequest, OverrideService, OverridePermissionChecker,
    ChangeWindowAuditEntry,
)

# Create override request for blocked deployment
request = EmergencyOverrideRequest(
    entity_id="entity-123",
    environment_id="production",
    requester="ops@team.com",
    justification="Critical hotfix for CVE-2024-XXX",
    blocking_rules=["Production Maintenance"],
)

# Approve with TTL
request.approve(approver="sre@team.com", expires_in=timedelta(hours=4))

# Use in constraint evaluation (evaluate_with_override checks for active overrides)
result = constraint.evaluate_with_override(context)

# Audit trail
entry = ChangeWindowAuditEntry(
    entity_id="entity-123",
    environment_id="production",
    evaluation_result=False,
    blocking_rules=["Outside change window"],
    override_used=True,
    override_id=request.id,
)
```

## Spoke Agent Setup

```python
from apollo.spoke import SpokeAgent, SpokeAgentConfig

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

# Auth (V3 — project-scoped)
apollo auth login
apollo auth whoami

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

2. **Plans require approval in production**: The `AccreditationRouter` enforces approval workflows. Bypass only with explicit `EmergencyOverrideRequest` and audit trail

3. **Constraints are AND-ed**: All constraints must be satisfied for a plan to proceed. A single violation blocks execution. Check `ConstraintResult.severity` — only `BLOCKING` prevents execution

4. **Constraints take context dicts, not Plan objects**: All constraint `evaluate()` methods take `context: dict[str, Any]`, not a `Plan`. Each constraint type documents its expected context keys

5. **Suppression windows are absolute**: Unlike maintenance windows (structured time ranges), suppression windows are one-time date ranges that block all changes. Set `allows_rollback=True` to exempt rollbacks

6. **V3 auth is project-scoped, not team-scoped**: Casbin domain is `project_id`. Use `create_project_with_admin()` and `add_project_member()` for setup. Old `Enforcer`, `Team`, `Role` classes no longer exist — use `ApolloEnforcer`, `create_enforcer()`, `require_operation()`

7. **Operations follow service:action-resource convention**: e.g., `catalog:read-product`, `orchestration:execute-plan`. Use `OperationRegistry.seed(SEED_OPERATIONS)` to bootstrap

8. **RIDs use Palantir format**: `ri.<service>.<instance>.<type>.<locator>`. Use typed generators like `generate_user_rid()` for correct format

9. **Release recall strategies matter**: `freeze` just prevents new installs, `warn` allows with notification, `force_rollback` actively rolls back running instances

10. **SpokeAgent needs network access**: The agent communicates with Hub via HTTP/WebSocket. Offline mode queues commands in Redis but requires eventual connectivity

11. **SQLModel persistence**: The `ProductCatalog` uses SQLModel with SQLite by default. Pass a connection string for PostgreSQL in production

12. **Events are synchronous by default**: Blinker signals fire synchronously. Long-running event handlers block the caller. Use `init_handlers()` for built-in async handler setup

13. **Version ordering is SLS-specific**: `ReleaseVersion` implements SLS ordering rules. `1.0.0 < 1.0.0-rc1` is false — release candidates sort before releases

14. **Compass resolve_project is O(1)**: Used by `@require_resource_operation` to determine project ownership without walking FK chains. Register resources via `CompassService.register_resource()`

15. **MaintenanceWindow uses structured objects**: Not cron strings. Use `MaintenanceWindow(name=..., start_time=..., end_time=...)` — cron scheduling is handled separately by `RecurringScheduleService`
