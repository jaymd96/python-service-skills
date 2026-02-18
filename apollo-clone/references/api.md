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
    MaintenanceWindowConstraint,
    DependencyConstraint,
    SuppressionWindowConstraint,
)

evaluator = ConstraintEvaluator()
result = evaluator.evaluate(plan, constraints)
# result.satisfied: bool, result.violations: list[str]
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

## SpokeAgent

Agent for spoke-side (cluster-level) operations. Communicates with Hub via HTTP polling or WebSocket.

```python
from apollo.spoke.agent import SpokeAgent, SpokeAgentConfig

config = SpokeAgentConfig(
    hub_url="https://hub.example.com",
    namespace="default",
    poll_interval=30,           # seconds
    offline_queue_enabled=True, # Redis-backed offline queue
)
agent = SpokeAgent(config)
agent.start()
```

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

Blinker-based pub/sub event system.

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
)

@plan_failed.connect
def on_plan_failed(sender, plan_id, error, **kwargs):
    alert(f"Plan {plan_id} failed: {error}")

@entity_state_changed.connect
def on_state_change(sender, entity_id, old_state, new_state, **kwargs):
    log.info(f"Entity {entity_id}: {old_state} -> {new_state}")
```

## FastAPI Application

```python
from apollo.api.app import create_app

app = create_app()  # Returns FastAPI instance
```

### Route modules (33 routers)

Agents, auth, auth_mfa, auth_oidc, changes, channels, config_versions, dependencies, deployments, drift, entities, environments, maintenance, modules, pipelines, plans, products, releases, registries, teams, vulnerabilities, notifications, preferences, analytics, audit, api_keys, hub.

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

## CLI

Click + Rich CLI with extensive subcommands.

```python
from apollo.cli.main import cli, ApolloContext

# ApolloContext provides shared state:
# - output format (table, json, yaml)
# - verbose mode
# - auth session
```

### Command groups

`agent`, `analytics`, `auth`, `change`, `channel`, `config_version`, `configure`, `dependency`, `deployment`, `drift`, `entity`, `environment`, `helm`, `maintenance`, `module`, `notification`, `pipeline`, `plan`, `preference`, `product`, `publish`, `registry`, `release`, `spec`, `spoke`, `team`, `vulnerability`

Features: audit logging middleware (FedRAMP/SOC2), offline mode with command queuing, multiple output formats.
