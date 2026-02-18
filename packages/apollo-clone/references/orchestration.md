# Apollo-Clone — Orchestration Engine

> Part of the apollo-clone skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [Engine Overview](#engine-overview)
- [Constraint System](#constraint-system)
- [Plan Execution](#plan-execution)
- [Approval Workflows](#approval-workflows)
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
3. Evaluate constraints (maintenance window, dependencies, suppression)
4. If all constraints satisfied → Plan state = PENDING
5. If constraints violated → Plan state = BLOCKED (with reasons)
```

## Constraint System

Constraints are preconditions that must be satisfied before a plan can execute.

```python
from apollo.orchestration import (
    Constraint, ConstraintResult, ConstraintEvaluator,
)

class Constraint:
    def evaluate(self, plan: Plan) -> ConstraintResult: ...

class ConstraintResult:
    satisfied: bool
    violations: list[str]
    metadata: dict
```

### Built-in constraints

#### MaintenanceWindowConstraint

Only allows execution during configured time windows.

```python
from apollo.orchestration import MaintenanceWindowConstraint

constraint = MaintenanceWindowConstraint(
    cron_expression="0 2 * * 6",  # Saturdays at 2 AM
    duration_hours=4,
    timezone="UTC",
)
# Plan blocked outside Saturday 2:00-6:00 AM UTC
```

#### DependencyConstraint

Validates that upstream service dependencies are healthy and running the required version.

```python
from apollo.orchestration import DependencyConstraint

constraint = DependencyConstraint(catalog=catalog)
# Checks: are all dependencies in RUNNING state?
# Are dependency versions within declared compatible range?
```

#### SuppressionWindowConstraint

Blocks all changes during configured suppression periods (holiday freezes, incident response).

```python
from apollo.orchestration import SuppressionWindowConstraint

constraint = SuppressionWindowConstraint(
    start="2024-12-20T00:00:00Z",
    end="2025-01-02T00:00:00Z",
    reason="Holiday freeze",
)
```

### ConstraintEvaluator

Rule-engine based evaluation (safe — no exec/eval).

```python
from apollo.orchestration import ConstraintEvaluator

evaluator = ConstraintEvaluator()
result = evaluator.evaluate(plan, [
    maintenance_window,
    dependency_check,
    suppression_check,
])
# result.satisfied == True only if ALL constraints pass
# result.violations lists all failing constraint reasons
```

## Plan Execution

### Plan lifecycle

```
PENDING → (approve) → EXECUTING → SUCCEEDED
                              ↘ FAILED → ROLLED_BACK
     ↘ BLOCKED (constraints not met)
          ↘ CANCELLED
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

### Plan execution dispatches to the SpokeAgent which uses `HelmChartOperator` for Helm-based deployments.

## Approval Workflows

### AccreditationRouter

Routes plans through multi-level approval workflows based on criticality, environment, and plan type.

```python
from apollo.orchestration import AccreditationRouter

router = AccreditationRouter(rules=[
    # Production upgrades require SRE approval
    ApprovalRule(
        environment="production",
        plan_types=[PlanType.UPGRADE],
        required_roles=["sre"],
    ),
    # Critical services need VP approval
    ApprovalRule(
        criticality=Criticality.CRITICAL,
        required_roles=["vp-engineering"],
    ),
])
```

### ApprovalGate

Individual gate requiring specific approvals.

```python
from apollo.orchestration import ApprovalGate

gate = ApprovalGate(
    required_approvers=2,
    required_roles=["sre", "product-owner"],
    ttl=timedelta(hours=24),  # Approval expires
)
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
    cron="0 2 * * 6",        # Saturdays at 2 AM
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

# InMemory — for tests
storage = InMemoryPlanStorage()

# Filesystem — for development
storage = FilesystemPlanStorage(path="/var/apollo/plans")

# Database — for production
storage = DatabasePlanStorage(catalog=catalog)
```

### Plan analytics

Track plan execution metrics:
- Success/failure rates by plan type
- Average execution duration
- Constraint violation frequency
- Rollback rates

## Spoke Agent Execution

When a plan reaches `EXECUTING`, it's dispatched to the appropriate SpokeAgent.

```
Hub (OrchestrationEngine)
  → Sends plan to SpokeAgent (HTTP/WebSocket)
    → SpokeAgent dispatches to HelmChartOperator
      → Helm install/upgrade/rollback
        → Agent reports result back to Hub
          → Entity state updated
```

### HelmChartOperator

Executes Helm operations based on plan type:

| PlanType | Helm Operation |
|----------|---------------|
| `install` | `helm install` |
| `upgrade` | `helm upgrade` |
| `rollback` | `helm rollback` |
| `uninstall` | `helm uninstall` |
| `config_update` | `helm upgrade` (values only) |

### ExpectedStateK8s

Reports actual Kubernetes state (pod status, resource usage, health) back to the Hub for reconciliation.
