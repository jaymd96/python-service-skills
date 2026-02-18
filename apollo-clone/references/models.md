# Apollo-Clone — Data Models

> Part of the apollo-clone skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [Product](#product)
- [Release](#release)
- [Entity](#entity)
- [EntityState FSM](#entitystate-fsm)
- [Plan](#plan)
- [PlanType & PlanState](#plantype--planstate)
- [Constraints](#constraints)
- [RBAC](#rbac)

## Product

```python
from apollo.models import Product, ProductId, ProductType

product = Product(
    group_id="com.example",
    artifact_id="my-service",
    name="My Service",
    product_type=ProductType.HELM_V1,
)
```

### ProductId

Maven coordinates: `group_id:artifact_id`

### ProductType

| Value | Description |
|-------|-------------|
| `helm.v1` | Python service wrapped in Helm chart |
| `asset.v1` | Static assets |
| `service.v1` | Standalone service |

### Related models

| Model | Description |
|-------|-------------|
| `ProductDependency` | Version constraints between products |
| `ProductIncompatibility` | Bidirectional incompatibility declarations |
| `ProductConfiguration` | Schema, defaults, and secrets |
| `ProductManifest` | Immutable metadata for releases |

## Release

Versioned software artifact with status tracking.

```python
from apollo.models import Release, ReleaseType, ReleaseStatus, ReleaseVersion

release = Release(
    product_id="com.example:my-service",
    version="1.0.0",
    release_type=ReleaseType.RELEASE,
    channel="stable",
)
```

### ReleaseType

| Value | Description |
|-------|-------------|
| `release` | Stable release |
| `release_candidate` | RC build |
| `snapshot` | Development snapshot |
| `rc_snapshot` | RC + development |

### ReleaseStatus

| Value | Description |
|-------|-------------|
| `active` | Available for deployment |
| `recalled` | Pulled from distribution |
| `deprecated` | Superseded by newer version |

### RecallStrategy

| Value | Description |
|-------|-------------|
| `freeze` | Block new installations |
| `warn` | Allow with warning |
| `force_rollback` | Force rollback running instances |

### ReleaseVersion

Parsed version with comparison and ordering support. `VersionMatcher` supports patterns like `1.x.x` for range matching.

## Entity

A deployed instance of a product in an environment.

```python
from apollo.models import Entity, EntityState, EntityHealth, Criticality

entity = Entity(
    product_id="com.example:my-service",
    environment="production",
    state=EntityState.UNMANAGED,
    health=EntityHealth.UNKNOWN,
    criticality=Criticality.HIGH,
)
```

### EntityHealth

| Value | Description |
|-------|-------------|
| `healthy` | All checks passing |
| `unhealthy` | One or more checks failing |
| `unknown` | Health not yet determined |

### Criticality

| Value | Description |
|-------|-------------|
| `low` | Best-effort SLA |
| `standard` | Default SLA |
| `high` | Priority SLA |
| `critical` | Highest priority |

### Related models

| Model | Description |
|-------|-------------|
| `EntitySettings` | Configuration for Apollo management |
| `ReportedState` | Current actual state from spoke agents |
| `DegradationReason` | Structured diagnostic codes for degraded entities |

## EntityState FSM

```
UNMANAGED → PENDING → INSTALLING → RUNNING ↔ DEGRADED
                                       ↘ FAILED
```

### State transitions

| From | To | Trigger |
|------|-----|---------|
| `UNMANAGED` | `PENDING` | Entity registered with Apollo |
| `PENDING` | `INSTALLING` | Plan execution begins |
| `INSTALLING` | `RUNNING` | Installation succeeds |
| `INSTALLING` | `FAILED` | Installation fails |
| `RUNNING` | `DEGRADED` | Health checks failing |
| `DEGRADED` | `RUNNING` | Health recovers |
| `RUNNING` | `FAILED` | Fatal error |
| `DEGRADED` | `FAILED` | Degradation worsens |

`EntityStateMachine` manages lifecycle transitions and validates state changes.

### Resource metrics

Entities track resource metrics with historical data for capacity planning. Rollback configuration and tracking support automated recovery.

## Plan

A unit of deployment work with constraint evaluation.

```python
from apollo.models import Plan, PlanType, PlanState

plan = Plan(
    entity_id="entity-123",
    plan_type=PlanType.UPGRADE,
    target_version="2.0.0",
    state=PlanState.PENDING,
)
```

## PlanType & PlanState

### PlanType

| Value | Description |
|-------|-------------|
| `install` | New installation |
| `upgrade` | Version upgrade |
| `config_update` | Configuration change only |
| `uninstall` | Remove entity |
| `rollback` | Revert to previous version |
| `secret_change` | Secret rotation |
| `custom` | Custom operation |

### PlanState

| Value | Description |
|-------|-------------|
| `pending` | Awaiting approval/execution |
| `blocked` | Constraints not satisfied |
| `executing` | Currently running |
| `succeeded` | Completed successfully |
| `failed` | Execution failed |
| `rolled_back` | Rolled back after failure |

`PlanStateMachine` manages plan lifecycle transitions.

## Constraints

Constraints are preconditions evaluated before plan execution.

### MaintenanceWindowConstraint

```python
from apollo.models import MaintenanceWindowConstraint

constraint = MaintenanceWindowConstraint(
    cron_expression="0 2 * * 6",  # Saturdays at 2 AM
    duration_hours=4,
)
```

### DependencyConstraint

Validates that service dependencies are healthy before proceeding.

### SuppressionConstraint

Blocks changes during configured suppression periods (e.g., holiday freezes).

## RBAC

Casbin-based authorization with hierarchical roles.

```python
from apollo.auth import Enforcer, Team, Role, Permission, ResourceType
```

### Role hierarchy

```
admin → editor → operator → viewer
```

### ResourceType

| Value | Description |
|-------|-------------|
| `environment` | Deployment environment |
| `product` | Product definition |
| `channel` | Release channel |
| `entity` | Deployed entity |
| `label` | Label-based access |

### Semantic roles

| Role | Scope |
|------|-------|
| `ProductRole` | Per-product permissions |
| `ChannelRole` | Per-channel permissions |
| `EnvironmentRole` | Per-environment permissions |
| `LabelRole` | Label-based permissions |

### Authentication features

| Feature | Description |
|---------|-------------|
| SAML integration | Enterprise SSO |
| OIDC support | OpenID Connect |
| MFA | TOTP, WebAuthn, recovery codes |
| API Keys | Prefixed keys with validation |
| Certificate management | Artifact signing |
| HSM integration | Hardware security modules |
| Vault integration | Encryption key management |
| Service accounts | Machine-to-machine auth |
| Compliance logging | FedRAMP/SOC2 audit trails |

### FastAPI decorators

```python
from apollo.auth import require_permission, require_role

@require_permission("product:write")
async def create_product(request): ...

@require_role("admin")
async def delete_environment(request): ...
```
