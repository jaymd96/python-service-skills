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
- [V3 Auth System](#v3-auth-system)
- [Compass & RIDs](#compass--rids)

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
UNMANAGED -> PENDING -> INSTALLING -> RUNNING <-> DEGRADED
                                          \-> FAILED
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
| `proposed` | Awaiting constraint evaluation |
| `blocked` | Constraints not satisfied |
| `issued` | Approved, ready for execution |
| `executing` | Currently running |
| `succeeded` | Completed successfully |
| `failed` | Execution failed |
| `rolled_back` | Rolled back after failure |

`PlanStateMachine` manages plan lifecycle transitions: `PROPOSED -> BLOCKED -> ISSUED -> EXECUTING -> SUCCEEDED/FAILED/ROLLED_BACK`.

## Constraints

Constraints are preconditions evaluated before plan execution. All constraints implement a uniform interface:

```python
from apollo.orchestration import (
    Constraint, ConstraintResult, ConstraintType, ConstraintSeverity,
)

# Base constraint: evaluate(context: dict[str, Any]) -> ConstraintResult
# See orchestration.md for full constraint system documentation
```

### ConstraintType (8 types)

| Value | Description |
|-------|-------------|
| `maintenance_window` | Time-window based deployment gates |
| `dependency` | Service dependency health/version checks |
| `suppression_window` | One-time deployment freezes |
| `approval` | Approval workflow gates |
| `health` | Prometheus-based health metric checks |
| `rate_limiting` | Deployment frequency limits |
| `version` | Version compatibility checks |
| `custom` | User-defined constraints |

### ConstraintSeverity

| Value | Description |
|-------|-------------|
| `blocking` | Plan cannot proceed |
| `warning` | Plan can proceed with acknowledgment |
| `info` | Informational only |

### ConstraintResult

```python
@dataclass
class ConstraintResult:
    satisfied: bool
    constraint_type: ConstraintType
    rule_expression: str
    context_data: dict[str, Any]
    message: str
    severity: ConstraintSeverity = ConstraintSeverity.BLOCKING
    evaluated_at: datetime  # UTC
    metadata: dict[str, Any] = field(default_factory=dict)
    reevaluate_at: datetime | None = None  # Used by ReevaluationScheduler
```

## V3 Auth System

The V3 auth redesign makes **Project** (not Team) the Casbin security domain. The auth module exports ~65 public symbols.

### 6-tuple permission model

```
(user_id, org_rid, project_id, resource_type, resource_id, action)
```

### Core imports

```python
from apollo.auth import (
    # Enforcer & factory
    ApolloEnforcer, create_enforcer, get_enforcer, clear_enforcer_cache,
    RBACStorageType,

    # Decorators (replace old @require_permission / @require_role)
    require_operation,           # e.g., @require_operation("catalog:read-product")
    require_resource_operation,  # V4: resolves project via Compass, then checks
    require_global_role,         # Non-project-scoped role checks

    # Authorization pipeline
    AuthorizationPipeline, AuthorizationRequest, AuthorizationResult,

    # Operations registry
    OperationRegistry, SEED_OPERATIONS,  # 70+ pre-defined operations

    # Auth service & chain
    AuthService, AuthChain, AuthMode, create_auth_service,

    # RBAC helpers
    can_view, can_edit, can_delete, can_admin, can_execute,
    create_project_with_admin, add_project_member, remove_project_member,
    DEFAULT_ORG_RID,
)
```

### ApolloEnforcer

Casbin wrapper with project-scoped RBAC:

```python
enforcer = create_enforcer(database_url="sqlite:///apollo.db")

# Core permission check
allowed = enforcer.check_permission(
    user_id="ri.auth.main.user.1234",
    org_rid="ri.auth.main.org.5678",
    project_id="project-456",
    resource_type="product",
    resource_id="com.example:my-service",
    action="write",
)

# With explanation (returns tuple[bool, list[str]])
allowed, reasons = enforcer.check_permission_with_explanation(...)

# Batch check
results = enforcer.batch_check_permissions([
    (user, org, project, "product", "*", "read"),
    (user, org, project, "entity", "entity-1", "write"),
])

# Role management (project-scoped)
enforcer.assign_role(user_id, "editor", project_id)
enforcer.get_user_roles(user_id, project_id)  # -> ["editor"]
enforcer.get_user_projects(user_id)  # -> ["project-456", ...]
```

### Authorization Pipeline (5 layers)

Evaluation order — short-circuits on first denial (except MFA):

| Layer | Name | Purpose |
|-------|------|---------|
| 1 | Cross-Org | Cross-organization trust evaluation |
| 2 | RBAC | Casbin permission check (project-scoped) |
| 3 | MAC | Mandatory access control (security markings) |
| 4 | MFA | Step-up authentication requirements |
| 5 | ABAC | Attribute-based access control (granular policies) |

```python
from apollo.auth import AuthorizationPipeline, AuthorizationRequest, AuthorizationResult

pipeline = AuthorizationPipeline(
    enforcer=enforcer,
    enable_cross_org=True,
    enable_mfa_rules=True,
)

request = AuthorizationRequest(
    user_id="ri.auth.main.user.1234",
    org_rid="ri.auth.main.org.5678",
    project_id="project-456",
    resource_type="product",
    resource_id="com.example:my-service",
    action="write",
    mfa_verified=True,
)

result = pipeline.evaluate(request)
# result.allowed: bool
# result.denied_by: str | None  ("cross_org", "rbac", "markings", "mfa", "granular")
# result.requires_mfa: bool
```

### Operations Registry

Data-driven, database-backed operations following `service:action-resource` convention:

```python
from apollo.auth import OperationRegistry, SEED_OPERATIONS

registry = OperationRegistry(session=db_session)
registry.seed(SEED_OPERATIONS)  # Seeds 70+ operations

# Examples of seed operations:
# ("catalog", "read-product", "View product details and releases", False)
# ("catalog", "delete-product", "Delete products", True)  # requires MFA
# ("orchestration", "execute-plan", "Execute deployment plans", False)

op = registry.get_by_qualified_name("catalog:read-product")
ops = registry.list_by_service("catalog")
```

### Decorators

```python
from apollo.auth import require_operation, require_resource_operation, require_global_role

# Operation-based (project-scoped)
@require_operation("catalog:read-product")
async def list_products(request): ...

# Resource-aware (resolves project via Compass)
@require_resource_operation(
    "catalog:write-product",
    resource_type="product",
    resource_param="product_id",
)
async def update_product(request, product_id: str): ...

# Global role (non-project-scoped)
@require_global_role("global_admin")
async def admin_operation(request): ...
```

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
| Break-glass bypass | Emergency super-admin via `APOLLO_BREAKGLASS_USERS` env var |
| Compliance logging | FedRAMP/SOC2 audit trails with structured `ChangeWindowAuditEntry` |

## Compass & RIDs

### Resource Identifiers (RIDs)

Palantir-style typed identifiers: `ri.<service>.<instance>.<type>.<locator>`

```python
from apollo.rid import ResourceIdentifier, generate_user_rid, generate_product_rid

# Generate typed RIDs
user_rid = generate_user_rid()      # -> "ri.auth.main.user.<uuid4>"
product_rid = generate_product_rid() # -> "ri.catalog.main.product.<uuid4>"

# Parse existing RIDs
rid = ResourceIdentifier.parse("ri.auth.main.user.abc123")
rid.service   # "auth"
rid.instance  # "main"
rid.type      # "user"
rid.locator   # "abc123"
```

18 typed aliases (compile-time safe via `NewType`):
`OrgRid`, `UserRid`, `GroupRid`, `TeamRid`, `SpaceRid`, `ProjectRid`, `FolderRid`, `ProductRid`, `ReleaseRid`, `EnvironmentRid`, `EntityRid`, `ChannelRid`, `PlanRid`, `ModuleRid`, `OperationRid`, `AuditEventRid`, `ServiceAccountRid`, `ApiKeyRid`

### Compass Resource Namespace

Unified registry providing RID paths, project ownership, and hierarchy traversal:

```
Organisation -> Space (governance) -> Project (security boundary) -> Folder -> Resource
```

```python
from apollo.compass import CompassService, CompassMode, create_compass_service

compass = create_compass_service(session=db_session, mode=CompassMode.LOCAL)

# Register resources in the hierarchy
compass.create_space(org_rid, name="Platform", slug="platform")
compass.create_project(space_rid, name="My Service", slug="my-service")
compass.register_resource(project_rid, resource_rid, "My Product", "product")

# Resolve project ownership (O(1) lookup — critical for V4 auth)
project = compass.resolve_project(resource_rid)
```
