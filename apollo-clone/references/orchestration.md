# Apollo-Clone — Orchestration Engine

> Part of the apollo-clone skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [Engine Overview](#engine-overview)
- [Constraint System](#constraint-system)
- [Built-in Constraints](#built-in-constraints)
- [Override System](#override-system)
- [Plan Execution](#plan-execution)
- [Approval Workflows](#approval-workflows)
- [Analytics & Metrics](#analytics--metrics)
- [Scheduling](#scheduling)
- [Plan Storage](#plan-storage)
- [Spoke Agent Execution](#spoke-agent-execution)

## Engine Overview

The `OrchestrationEngine` is the central decision-maker. It evaluates the desired state vs. actual state of entities and proposes plans to reconcile them.

```python
from apollo.orchestration import OrchestrationEngine

engine = OrchestrationEngine(
    catalog=catalog,
    plan_storage=storage,
)

# Evaluate a single entity
plans = engine.evaluate(entity)

# Evaluate all entities
plans = engine.evaluate_all()

# Execute a plan
result = engine.execute_plan(plan_id)
```

### Evaluation flow

```
1. Compare desired state vs. reported state
2. Determine plan type (install, upgrade, rollback, etc.)
3. Evaluate constraints (maintenance window, dependencies, suppression, quorum, health, rate limits)
4. If all constraints satisfied -> Plan state = PROPOSED -> ISSUED
5. If constraints violated -> Plan state = BLOCKED (with reasons + reevaluate_at)
```

### Orchestration exports

```python
from apollo.orchestration import (
    # Engine
    OrchestrationEngine, EvaluationResult, PlanProposal, ExecutionMode,

    # Constraint system
    Constraint, ConstraintResult, ConstraintType, ConstraintSeverity,
    ConstraintEvaluator, RuleCache,

    # Built-in constraints
    MaintenanceWindowConstraint, MaintenanceWindow,
    DependencyConstraint, DependencyRequirement, DependencyStatus, ServiceHealth,
    SuppressionWindowConstraint, SuppressionWindow, SuppressionType, SuppressionSource,
    QuorumConstraint,  # QuorumRequirement from apollo.spec
    NetworkPolicyConstraint,
    HealthConstraint,
    RateLimitConstraint, RateLimit, RateLimitScope,

    # Overrides
    OverrideService, OverridePermissionChecker, InMemoryOverrideStorage,
    EmergencyOverrideRequest, EmergencyOverrideStatus, ChangeWindowAuditEntry,
    ChangeWindow, DayOfWeek,

    # Analytics
    PlanAnalytics, PlanMetrics, PeriodType, TrendDirection,

    # Approval
    AccreditationRouter, ApprovalGate,

    # Scheduling
    ReevaluationScheduler, RecurringScheduleService,

    # Storage
    PlanStorage, InMemoryPlanStorage, FilesystemPlanStorage, DatabasePlanStorage,

    # CRD execution
    CRDExecutor,
)
```

## Constraint System

All constraints implement a uniform interface. The `evaluate()` method takes a context dict (not a Plan object) and returns a `ConstraintResult`.

### Base class

```python
class Constraint(ABC):
    def __init__(
        self,
        name: str,
        description: str | None = None,
        severity: ConstraintSeverity = ConstraintSeverity.BLOCKING,
        allow_override: bool = True,
    ) -> None: ...

    @property
    @abstractmethod
    def constraint_type(self) -> ConstraintType: ...

    @abstractmethod
    def evaluate(self, context: dict[str, Any]) -> ConstraintResult: ...

    def validate_expression(self, expression: str) -> tuple[bool, str | None]: ...

    def check_override(self, context: dict[str, Any]) -> tuple[bool, dict[str, Any] | None]: ...

    def evaluate_with_override(self, context: dict[str, Any]) -> ConstraintResult:
        """Evaluate with override checking — if an active override exists, returns satisfied."""
```

### ConstraintResult

```python
@dataclass
class ConstraintResult:
    satisfied: bool
    constraint_type: ConstraintType
    rule_expression: str            # The rule-engine expression evaluated
    context_data: dict[str, Any]    # Data dict used for evaluation
    message: str                    # Human-readable explanation
    severity: ConstraintSeverity = ConstraintSeverity.BLOCKING
    evaluated_at: datetime          # UTC timestamp
    metadata: dict[str, Any] = field(default_factory=dict)
    reevaluate_at: datetime | None = None  # For ReevaluationScheduler

    def is_blocking(self) -> bool: ...
    def to_dict(self) -> dict[str, Any]: ...
```

### ConstraintType & ConstraintSeverity

```python
class ConstraintType(Enum):
    MAINTENANCE_WINDOW = "maintenance_window"
    DEPENDENCY = "dependency"
    SUPPRESSION_WINDOW = "suppression_window"
    APPROVAL = "approval"
    HEALTH = "health"
    RATE_LIMITING = "rate_limiting"
    VERSION = "version"
    CUSTOM = "custom"

class ConstraintSeverity(Enum):
    BLOCKING = "blocking"    # Plan cannot proceed
    WARNING = "warning"      # Plan can proceed with acknowledgment
    INFO = "info"            # Informational only
```

### ConstraintEvaluator

Rule-engine based evaluation (safe — no exec/eval):

```python
from apollo.orchestration import ConstraintEvaluator

evaluator = ConstraintEvaluator()
result = evaluator.evaluate(context, [
    maintenance_constraint,
    dependency_constraint,
    suppression_constraint,
])
# result.satisfied == True only if ALL constraints pass
```

## Built-in Constraints

### MaintenanceWindowConstraint

Only allows execution during configured maintenance windows. Uses rule-engine expressions for evaluation.

```python
from apollo.orchestration import MaintenanceWindowConstraint, MaintenanceWindow

window = MaintenanceWindow(
    name="Saturday maintenance",
    start_time=datetime(2024, 12, 7, 2, 0, tzinfo=UTC),
    end_time=datetime(2024, 12, 7, 6, 0, tzinfo=UTC),
    affected_services=["catalog-service"],  # empty = all services
    allows_downtime=False,
    description="Weekly maintenance window",
    created_by="ops@team.com",
)

constraint = MaintenanceWindowConstraint(
    name="Production Maintenance",
    windows=[window],
    require_downtime_window=True,
    maintenance_override_service=override_service,  # optional
)

# Context keys: target_service, requires_downtime, current_time, environment_id
result = constraint.evaluate({"target_service": "catalog-service"})
```

**Rule expressions** (predefined):
- `WITHIN_WINDOW`: `$now >= window_start and $now < window_end`
- `WINDOW_STARTING_SOON`: `$now >= window_start - t"PT30M" and $now < window_start`
- `WINDOW_ACTIVE_FOR_SERVICE`: checks window + service membership
- `OVERRIDE_ACTIVE`: checks maintenance override status

### DependencyConstraint

Validates that upstream service dependencies are healthy and running required versions.

```python
from apollo.orchestration import (
    DependencyConstraint, DependencyRequirement, DependencyStatus, ServiceHealth,
)

requirement = DependencyRequirement(
    service_name="auth-service",
    min_version="2.0.0",
    required_health=ServiceHealth.HEALTHY,
    is_critical=True,
)

constraint = DependencyConstraint(
    name="Service Dependencies",
    requirements=[requirement],
    allow_degraded=False,  # If True, accepts DEGRADED as passing
)

# Context keys: dependencies (list of DependencyStatus objects)
result = constraint.evaluate({
    "dependencies": [
        DependencyStatus(
            name="auth-service",
            status=ServiceHealth.HEALTHY,
            version="2.1.0",
            min_required_version="2.0.0",
            is_critical=True,
        ),
    ],
})
```

**Rule expressions**: `IS_HEALTHY`, `IS_HEALTHY_OR_DEGRADED`, `VERSION_SATISFIED`, `FULL_CHECK`, `CRITICAL_ONLY`

### SuppressionWindowConstraint

Blocks changes during configured suppression periods (holiday freezes, incident response, promotions).

```python
from apollo.orchestration import (
    SuppressionWindowConstraint, SuppressionWindow,
    SuppressionType, SuppressionSource,
)

window = SuppressionWindow(
    name="Holiday freeze",
    start_time=datetime(2024, 12, 20, 0, 0, tzinfo=UTC),
    end_time=datetime(2025, 1, 2, 0, 0, tzinfo=UTC),
    suppression_type=SuppressionType.ALL,       # ALL, DEPLOYMENT, ALERT
    source=SuppressionSource.MANUAL,            # MANUAL, FAILURE, PROMOTION, POLICY
    reason="Holiday freeze",
    allows_rollback=True,                       # Rollbacks bypass suppression
)

constraint = SuppressionWindowConstraint(
    name="Holiday Suppression",
    windows=[window],
)

# Context keys: target_service, operation_type, is_rollback, current_time
result = constraint.evaluate({"target_service": "catalog-service", "is_rollback": False})
```

### QuorumConstraint

Validates replica availability during rolling updates and scale operations.

```python
from apollo.orchestration import QuorumConstraint
from apollo.spec import QuorumRequirement

constraint = QuorumConstraint(
    name="Replica Quorum",
    requirement=QuorumRequirement.ALL_BUT_ONE,
)

# Context keys: total_replicas, available_replicas, target_replicas, plan_type
result = constraint.evaluate({
    "total_replicas": 5,
    "available_replicas": 4,
    "plan_type": "upgrade",
})
```

### NetworkPolicyConstraint

Validates Rubix zero-trust network policies before plan execution.

```python
from apollo.orchestration import NetworkPolicyConstraint

constraint = NetworkPolicyConstraint(
    name="network-policy-constraint",
    require_policy=True,
    validate_syntax=True,
    check_dependencies=True,
    k3s_client=k3s_client,  # optional
)

# Context keys: entity_id, environment_id, network_policy, require_network_policy,
#               dependencies, dependency_policies
result = constraint.evaluate({
    "entity_id": "entity-123",
    "environment_id": "production",
    "network_policy": network_policy_spec,
})
```

### HealthConstraint

Verifies service health metrics via Prometheus integration.

```python
from apollo.orchestration import HealthConstraint
```

Integrates with `PrometheusClient` for real-time metric queries. Supports configurable thresholds (`ThresholdType`: `GREATER_THAN`, `LESS_THAN`, `BETWEEN`, etc.).

### RateLimitConstraint

Enforces deployment rate limits at entity, environment, or global scope.

```python
from apollo.orchestration import RateLimitConstraint, RateLimit, RateLimitScope

limit = RateLimit(
    max_deployments=5,
    time_window=timedelta(hours=1),
    scope=RateLimitScope.ENVIRONMENT,  # ENTITY, ENVIRONMENT, GLOBAL
    exclude_types=["rollback"],         # Rollbacks exempt
)

constraint = RateLimitConstraint(
    name="Deployment Rate Limit",
    limits=[limit],
)

# Context keys: entity_id, environment_id, plan_type, current_time
result = constraint.evaluate({
    "entity_id": "entity-123",
    "environment_id": "production",
    "plan_type": "upgrade",
})
```

## Override System

Emergency overrides allow bypassing constraints with audit trails.

### EmergencyOverrideRequest

```python
from apollo.orchestration import EmergencyOverrideRequest, EmergencyOverrideStatus

request = EmergencyOverrideRequest(
    entity_id="entity-123",
    environment_id="production",
    requester="ops@team.com",
    justification="Critical hotfix for CVE-2024-XXX",
    blocking_rules=["Production Maintenance"],
)

# Lifecycle: PENDING -> APPROVED -> USED (or DENIED/EXPIRED)
request.approve(approver="sre@team.com", expires_in=timedelta(hours=4))
request.mark_used()
request.is_valid()  # checks approved + not expired
```

### OverrideService

```python
from apollo.orchestration import OverrideService, OverridePermissionChecker

checker = OverridePermissionChecker(enforcer=apollo_enforcer)
service = OverrideService(permission_checker=checker)

# Permission checks
checker.can_request_override(user_id, team_id, environment_id, override_type)
checker.can_approve_override(user_id, team_id, environment_id, override_type)
checker.can_revoke_override(user_id, team_id, environment_id)
```

### Change Windows (Downtime Rules)

```python
from apollo.orchestration import ChangeWindow, DayOfWeek

window = ChangeWindow(
    name="Weekday deployments",
    days=[DayOfWeek.MONDAY, DayOfWeek.TUESDAY, DayOfWeek.WEDNESDAY,
          DayOfWeek.THURSDAY, DayOfWeek.FRIDAY],
    start_time=time(9, 0),
    end_time=time(17, 0),
    timezone="US/Eastern",
    blackout_dates=[datetime(2024, 12, 25)],
    allow_emergency_override=True,
    emergency_approver_roles=["admin", "incident_commander"],
)

window.is_active()                # Check if currently within window
window.next_window_start()        # Calculate next opening
```

### Audit Trail

```python
from apollo.orchestration import ChangeWindowAuditEntry

entry = ChangeWindowAuditEntry(
    entity_id="entity-123",
    environment_id="production",
    evaluation_result=False,
    blocking_rules=["Outside change window"],
    override_used=True,
    override_id="override-456",
    requester="ops@team.com",
)
entry.to_json()  # Structured audit record
```

## Plan Execution

### Plan lifecycle

```
PROPOSED -> (constraints) -> BLOCKED (reevaluate_at)
                          -> ISSUED -> (approve) -> EXECUTING -> SUCCEEDED
                                                             -> FAILED -> ROLLED_BACK
         -> CANCELLED
```

### Execution steps

```python
# Approve a pending plan
engine.approve_plan(plan_id, approved_by="ops@team.com")

# Execute (after approval if required)
result = engine.execute_plan(plan_id)

# Cancel
engine.cancel_plan(plan_id, reason="No longer needed")
```

### CRD Executor

For Kubernetes-native plan execution:

```python
from apollo.orchestration import CRDExecutor, ExecutionMode

executor = CRDExecutor(mode=ExecutionMode.CRD)
```

## Approval Workflows

### AccreditationRouter

Routes plans through multi-level approval workflows based on criticality, environment, and plan type.

```python
from apollo.orchestration import AccreditationRouter

router = AccreditationRouter(rules=[
    ApprovalRule(
        environment="production",
        plan_types=[PlanType.UPGRADE],
        required_roles=["sre"],
    ),
    ApprovalRule(
        criticality=Criticality.CRITICAL,
        required_roles=["vp-engineering"],
    ),
])
```

### ApprovalGate

```python
from apollo.orchestration import ApprovalGate

gate = ApprovalGate(
    required_approvers=2,
    required_roles=["sre", "product-owner"],
    ttl=timedelta(hours=24),
)
```

## Analytics & Metrics

### PlanAnalytics

```python
from apollo.orchestration import PlanAnalytics, PlanMetrics, PeriodType

analytics = PlanAnalytics(storage=plan_storage)

# Aggregated metrics
metrics = analytics.get_metrics(
    period_type=PeriodType.WEEKLY,
    start_date=date(2024, 12, 1),
    end_date=date(2024, 12, 31),
    environment_id="production",
)

for m in metrics:
    print(f"Week: {m.period_start} - Success: {m.success_rate:.1f}%")
    print(f"  Total: {m.total_plans}, Failed: {m.failed_plans}")
    print(f"  P50 duration: {m.duration_p50_seconds}s")
    print(f"  Constraint violations: {m.constraint_violation_count}")
```

### PlanMetrics

```python
@dataclass
class PlanMetrics:
    period_type: PeriodType            # DAILY, WEEKLY, MONTHLY
    period_start: datetime
    period_end: datetime
    environment_id: str | None
    total_plans: int
    successful_plans: int
    failed_plans: int
    cancelled_plans: int
    blocked_plans: int
    duration_p50_seconds: float | None
    duration_p95_seconds: float | None
    duration_p99_seconds: float | None
    avg_constraints_evaluated: float | None
    constraint_violation_count: int

    @property
    def success_rate(self) -> float: ...
    @property
    def failure_rate(self) -> float: ...
    @property
    def completion_rate(self) -> float: ...
```

### Trend Analysis

```python
trends = analytics.get_trends(
    metric="success_rate",
    period_type=PeriodType.DAILY,
    periods=30,
    environment_id="production",
)
# TrendDirection: INCREASING, DECREASING, STABLE
```

## Scheduling

### ReevaluationScheduler

Periodically re-evaluates entity state to detect drift and propose corrective plans.

```python
from apollo.orchestration import ReevaluationScheduler

scheduler = ReevaluationScheduler(
    engine=engine,
    interval=timedelta(minutes=5),
)
scheduler.start()
```

### RecurringScheduleService

Cron-based maintenance window scheduling.

```python
from apollo.orchestration import RecurringScheduleService

service = RecurringScheduleService(catalog=catalog)
service.create_schedule(
    name="weekly-maintenance",
    cron="0 2 * * 6",
    duration_hours=4,
    environments=["production"],
)
```

## Plan Storage

Three storage backends for plan persistence:

```python
from apollo.orchestration import (
    InMemoryPlanStorage,      # Testing
    FilesystemPlanStorage,    # Development
    DatabasePlanStorage,      # Production (SQLModel)
)

storage = InMemoryPlanStorage()
storage = FilesystemPlanStorage(path="/var/apollo/plans")
storage = DatabasePlanStorage(catalog=catalog)
```

## Spoke Agent Execution

When a plan reaches `EXECUTING`, it's dispatched to the appropriate SpokeAgent.

```
Hub (OrchestrationEngine)
  -> Sends plan to SpokeAgent (HTTP/WebSocket)
    -> SpokeAgent dispatches to HelmChartOperator
      -> Helm install/upgrade/rollback
        -> Agent reports result back to Hub
          -> Entity state updated
```

### HelmChartOperator

| PlanType | Helm Operation |
|----------|---------------|
| `install` | `helm install` |
| `upgrade` | `helm upgrade` |
| `rollback` | `helm rollback` |
| `uninstall` | `helm uninstall` |
| `config_update` | `helm upgrade` (values only) |

### ExpectedStateK8s

Reports actual Kubernetes state (pod status, resource usage, health) back to the Hub for reconciliation.
