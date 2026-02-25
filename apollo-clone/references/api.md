# Apollo-Clone — Core API Reference

> Part of the apollo-clone skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [ProductCatalog](#productcatalog)
- [OrchestrationEngine](#orchestrationengine)
- [SpokeAgent](#spokeagent)
- [Events](#events)
- [FastAPI Application](#fastapi-application)
- [CLI](#cli)

## ProductCatalog

High-level interface for product, release, entity, and plan CRUD. SQLModel-based persistence with SQLite backend.

```python
from apollo import ProductCatalog

catalog = ProductCatalog(":memory:")  # or path to SQLite DB
```

### Product operations

```python
catalog.create_product(product: Product) -> Product
catalog.get_product(product_id: str) -> Product | None
catalog.list_products() -> list[Product]
catalog.update_product(product: Product) -> Product
catalog.delete_product(product_id: str) -> None
```

### Release operations

```python
catalog.create_release(release: Release) -> Release
catalog.get_release(release_id: str) -> Release | None
catalog.list_releases(product_id: str, channel: str = None) -> list[Release]
catalog.promote_release(release_id: str, target_channel: str) -> Release
catalog.deprecate_release(release_id: str, reason: str) -> Release
catalog.recall_release(release_id: str, strategy: RecallStrategy) -> Release
```

### Entity operations

```python
catalog.create_entity(entity: Entity) -> Entity
catalog.get_entity(entity_id: str) -> Entity | None
catalog.list_entities(environment: str = None) -> list[Entity]
catalog.update_entity(entity: Entity) -> Entity
catalog.transition_entity(entity_id: str, target_state: EntityState) -> Entity
```

### Plan operations

```python
catalog.create_plan(plan: Plan) -> Plan
catalog.get_plan(plan_id: str) -> Plan | None
catalog.list_plans(state: PlanState = None) -> list[Plan]
```

### Environment & Channel operations

```python
catalog.create_environment(env: Environment) -> Environment
catalog.list_environments() -> list[Environment]
catalog.create_release_channel(channel: ReleaseChannel) -> ReleaseChannel
catalog.list_release_channels() -> list[ReleaseChannel]
```

### Catalog support services

| Component | Description |
|-----------|-------------|
| `DependencyGraph` | Product dependency resolution |
| `ProductResolver` | Dependency resolution with cycle detection |
| `ReleasePromotionWorkflow` | Promotion pipeline execution |
| `ModuleRegistry` | Module registration and lookup |
| `ReleaseNotesGenerator` | Automated release notes |
| `ExportService` / `ImportService` | Data import/export |

## OrchestrationEngine

Central engine that evaluates entities and proposes deployment plans.

```python
from apollo.orchestration import OrchestrationEngine

engine = OrchestrationEngine(catalog=catalog, plan_storage=storage)
```

### Core methods

| Method | Description |
|--------|-------------|
| `evaluate(entity) -> list[Plan]` | Evaluate entity and propose plans |
| `evaluate_all() -> list[Plan]` | Evaluate all entities |
| `execute_plan(plan_id) -> PlanResult` | Execute a plan |
| `approve_plan(plan_id, approved_by) -> Plan` | Approve a pending plan |
| `cancel_plan(plan_id, reason) -> Plan` | Cancel a plan |

### Constraint evaluation

```python
from apollo.orchestration import (
    Constraint, ConstraintResult, ConstraintEvaluator,
    MaintenanceWindowConstraint, MaintenanceWindow,
    DependencyConstraint, DependencyRequirement,
    SuppressionWindowConstraint, SuppressionWindow,
    QuorumConstraint, NetworkPolicyConstraint,
    HealthConstraint, RateLimitConstraint,
    ConstraintType, ConstraintSeverity,
)

# All constraints use uniform interface:
# constraint.evaluate(context: dict[str, Any]) -> ConstraintResult

evaluator = ConstraintEvaluator()
result = evaluator.evaluate(context, constraints)
```

### ConstraintResult

```python
@dataclass
class ConstraintResult:
    satisfied: bool
    constraint_type: ConstraintType       # 8 types: MAINTENANCE_WINDOW, DEPENDENCY, etc.
    rule_expression: str                  # The rule-engine expression evaluated
    context_data: dict[str, Any]          # Data dict used for evaluation
    message: str                          # Human-readable explanation
    severity: ConstraintSeverity          # BLOCKING, WARNING, INFO
    evaluated_at: datetime                # UTC
    metadata: dict[str, Any]
    reevaluate_at: datetime | None        # For ReevaluationScheduler
```

### Orchestration exports

```python
from apollo.orchestration import (
    # Engine
    OrchestrationEngine, EvaluationResult, ExecutionMode,

    # Constraints (8 built-in types)
    MaintenanceWindowConstraint, DependencyConstraint,
    SuppressionWindowConstraint, QuorumConstraint,
    NetworkPolicyConstraint, HealthConstraint, RateLimitConstraint,

    # Override system
    OverrideService, OverridePermissionChecker,
    EmergencyOverrideRequest, EmergencyOverrideStatus,
    ChangeWindow, ChangeWindowAuditEntry,

    # Analytics
    PlanAnalytics, PlanMetrics, PeriodType, TrendDirection,

    # CRD execution
    CRDExecutor,
)
```

### Plan storage backends

```python
from apollo.orchestration import (
    InMemoryPlanStorage,
    FilesystemPlanStorage,
    DatabasePlanStorage,
)
```

### Advanced features

| Component | Description |
|-----------|-------------|
| `AccreditationRouter` | Routes plans through multi-level approval workflows |
| `ApprovalGate` | Gate requiring specific approvals before execution |
| `ReevaluationScheduler` | Periodic re-evaluation of entity state |
| `RecurringScheduleService` | Cron-based maintenance window scheduling |
| `OverrideService` | Emergency override lifecycle management |
| `PlanAnalytics` | Plan metrics aggregation and trend analysis |
| `CRDExecutor` | Kubernetes CRD-based plan execution |

## SpokeAgent

Agent for spoke-side (cluster-level) operations. Communicates with Hub via HTTP polling or WebSocket.

```python
from apollo.spoke import SpokeAgent, SpokeAgentConfig

config = SpokeAgentConfig(
    hub_url="https://hub.example.com",
    namespace="default",
    poll_interval=30,           # seconds
    offline_queue_enabled=True, # Redis-backed offline queue
)
agent = SpokeAgent(config)
agent.start()
```

### Spoke components

| Component | Description |
|---------|-------------|
| `SpokeAgent` / `SpokeAgentConfig` | Agent orchestrator and configuration |
| `HelmChartOperator` / `HelmPlanExecutor` | Helm-based plan execution |
| `NodeLifecycleController` | Node termination policies and lifecycle |
| `ApolloAuthBroker` | Authentication mediation for spoke-side services |
| `ExpectedStateK8s` | Reports actual K8s state back to Hub |

### Node status

```python
from apollo.spoke.agent import NodeStatus

# NodeStatus enum:
# initializing, ready, busy, error, disconnected
```

### Capabilities

| Feature | Description |
|---------|-------------|
| HTTP polling | Default communication mode |
| WebSocket | Real-time updates with reconnection |
| Heartbeat | Connection health monitoring |
| Offline queue | Redis-backed command queue for disconnected operation |
| HelmChartOperator | Dispatches plans to Helm for execution |
| ExpectedStateK8s | Reports actual state back to Hub |

## Events

Blinker-based pub/sub event system with 20+ signals.

```python
from apollo.events import (
    # Plan lifecycle
    plan_created, plan_validated, plan_approved,
    plan_executing, plan_step_completed,
    plan_completed, plan_failed, plan_cancelled,
    # Entity state
    entity_created, entity_updated, entity_deleted,
    entity_state_changed, entity_health_changed,
    # Overrides
    override_requested, override_approved,
    override_rejected, override_expired,
    override_revoked, override_used,
    # Promotions
    release_promoted,
    # System
    system_startup, system_shutdown, config_reloaded,
    # Handler management
    init_handlers, register_handler, unregister_handler,
)

@plan_failed.connect
def on_plan_failed(sender, plan_id, error, **kwargs):
    alert(f"Plan {plan_id} failed: {error}")

@entity_state_changed.connect
def on_state_change(sender, entity_id, old_state, new_state, **kwargs):
    log.info(f"Entity {entity_id}: {old_state} -> {new_state}")
```

### Event handler subsystem

| Component | Description |
|-----------|-------------|
| `handlers/notifications.py` | Notification facade |
| `handlers/_transports.py` | Transport adapters (Slack, email, etc.) |
| `handlers/_signal_handlers.py` | Signal-to-handler wiring |
| `handlers/_digest.py` | Digest/batched notifications |
| `handlers/_preferences.py` | Per-user notification preferences |

## FastAPI Application

```python
from apollo.api import create_app

app = create_app()  # Returns FastAPI instance
```

### API exports

```python
from apollo.api import (
    create_app,
    # Errors
    APIError, BadRequestError, ConflictError, NotFoundError,
    UnauthorizedError, PolicyViolationError, ValidationError,
    # Dependency injection
    get_db, get_catalog, get_enforcer, get_current_user,
    set_db_engine, set_catalog, set_enforcer,
    # Common classes
    CurrentUser, PaginationParams,
)
```

### Route organization (50+ routes across 27 routers)

**Core domain routes:**
- `products_router` — `/products`
- `releases_router` — `/releases` (includes auth, canary, promotion, demotion, approval)
- `environments_router` — `/environments` (includes config, CRUD, heartbeat, maintenance, manifests, metrics)
- `entities_router` — `/entities`
- `plans_router` — `/plans`
- `channels_router` — `/channels`
- `agents_router` — `/agents`
- `pipelines_router` — `/pipelines`
- `modules_router` — `/modules`

**Data & operations routes:**
- `changes_router`, `maintenance_router`, `api_keys_router`, `deployments_router`
- `notifications_router`, `preferences_router`, `analytics_router`, `registries_router`
- `drift_router`, `audit_router`, `config_versions_router`, `dependencies_router`
- `vulnerabilities_router`

**Organizational routes:**
- `groups_router` — `/groups`
- `teams_router` — `/teams`
- `projects_router` — `/projects`
- `spaces_router` — `/spaces`
- `hub_router` — `/hub`

**Auth routes (combined router):**
- `auth_combined_router` — `/auth` (auth + MFA + OIDC)

**Admin routes:**
- `admin_router` — `/admin` (attributes, IDP, markings, operations, policies, rules, trust)

**Protocol routes:**
- `protocol_router` — `/protocol` (identity, authorization, compass)

### Route subdirectories

| Directory | Modules | Purpose |
|-----------|---------|---------|
| `routes/` | 26 top-level modules | Core domain endpoints |
| `routes/admin/` | 8 modules | Admin management (IDP, markings, operations, policies, rules, trust) |
| `routes/auth/` | 3 modules | Authentication (auth, MFA, OIDC) |
| `routes/environments/` | 7 modules | Environment management (config, CRUD, heartbeat, maintenance, manifests, metrics) |
| `routes/protocol/` | 3 modules | Protocol endpoints (identity, authorization, compass) |
| `routes/releases/` | 7 modules | Release management (approval, auth, canary, CRUD, demotion, promotion) |

### API schemas (13 schema modules)

Located in `apollo/api/schemas/`: agents, changes, channels, common, deployments, entities, environments, maintenance, modules, plans, products, releases.

### Key endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/products` | List products |
| POST | `/api/v1/products` | Create product |
| GET | `/api/v1/products/{id}` | Get product |
| GET | `/api/v1/products/{id}/releases` | List releases |
| POST | `/api/v1/releases` | Create release |
| GET | `/api/v1/entities` | List entities |
| POST | `/api/v1/plans` | Create plan |
| POST | `/api/v1/plans/{id}/approve` | Approve plan |
| GET | `/api/v1/environments` | List environments |
| POST | `/admin/operations` | Manage operations registry |
| GET | `/protocol/compass/{rid}` | Compass resource lookup |
| POST | `/auth/mfa/verify` | MFA verification |

### Exception hierarchy

```python
APIError(status_code, message, details)  # Base
├── BadRequestError        # 400
├── UnauthorizedError      # 401
├── PolicyViolationError   # 403
├── NotFoundError          # 404
├── ConflictError          # 409
└── ValidationError        # 422
```

### Dependency injection (`_deps.py`)

| Function | Returns |
|----------|---------|
| `get_db()` | SQLModel session |
| `get_db_session()` | Async session |
| `get_catalog()` | `ProductCatalog` |
| `get_enforcer()` | `ApolloEnforcer` |
| `get_current_user()` | `CurrentUser` (from Bearer token or X-API-Key) |
| `get_channel_manager()` | `ChannelManager` |
| `get_auth_chain()` | `AuthChain` |
| `get_auth_service()` | `AuthService` |
| `get_module_registry()` | `ModuleRegistry` |
| `get_compass_service()` | `CompassService` |
| `get_dependency_graph()` | `DependencyGraph` |
| `get_audit_service()` | `AuditService` |
| `get_health_collector()` | `HealthCollector` |

## CLI

Click + Rich CLI with extensive subcommands and audit logging.

```python
from apollo.cli.main import cli, ApolloContext

# ApolloContext provides shared state:
# - output format (table, json, yaml)
# - verbose mode
# - auth session
```

### Command groups

`agent`, `analytics`, `auth`, `change`, `channel`, `config_version`, `configure`, `dependency`, `deployment`, `drift`, `entity`, `environment`, `helm`, `maintenance`, `module`, `notification`, `pipeline`, `plan`, `preference`, `product`, `publish`, `registry`, `release`, `spec`, `spoke`, `team`, `vulnerability`

### Features

- Audit logging middleware (FedRAMP/SOC2) with `CLIAuditLogger`
- Offline mode with command queuing
- Multiple output formats (table, json, yaml)
- `HubAPIClient` for all server communication (fully decoupled from server)
