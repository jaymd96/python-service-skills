# Citadel — Core API Reference

> Part of the citadel skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [NodeManager](#nodemanager)
- [HealthChecker](#healthchecker)
- [MultipassProvider](#multipassprovider)
- [Data Classes](#data-classes)
- [NodeTerminationPolicy CRD](#nodeterminationpolicy-crd)
- [Default Policies](#default-policies)

## NodeManager

Manages Kubernetes node lifecycle using the Eviction API (respects PodDisruptionBudgets).

```python
from rubix_lite.k3s.node import NodeManager

manager = NodeManager(kubeconfig=None)  # None = in-cluster config
```

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `list_nodes()` | `list[NodeInfo]` | List all nodes |
| `get_node(name)` | `NodeInfo` | Get single node info |
| `get_node_age_hours(name)` | `float` | Node age in hours |
| `cordon(name)` | None | Mark node unschedulable |
| `uncordon(name)` | None | Mark node schedulable |
| `label_for_termination(name, reason, policy_name)` | None | Label node with termination intent |
| `clear_termination_labels(name)` | None | Remove termination labels |
| `drain(name, timeout_seconds=300, delete_emptydir_data=True)` | `DrainResult` | Drain node via Eviction API |

### Drain operation flow

```
1. Cordon the node
2. List non-DaemonSet pods on the node
3. Evict each pod via Eviction API (respects PDB)
4. Retry on 429 (PDB blocking) with exponential backoff
5. Wait for pods to actually terminate
```

### Node labels used

| Label | Value |
|-------|-------|
| `rubix.dev/terminate-reason` | Why node is being terminated |
| `rubix.dev/terminate-after` | Timestamp of termination intent |
| `rubix.dev/policy-name` | Which policy triggered termination |
| `rubix.dev/managed-by` | `"rubix-lifecycle"` |

## HealthChecker

Read-only assessment layer that evaluates node health without taking action.

```python
from rubix_lite.k3s.health import HealthChecker, HealthState

checker = HealthChecker(max_age_hours=48.0)
```

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `check(node: NodeInfo)` | `NodeHealthReport` | Health report for single node |
| `check_all(nodes: list[NodeInfo])` | `list[NodeHealthReport]` | Reports for all nodes |

### HealthState Enum

Ordered by severity:

| State | Meaning |
|-------|---------|
| `HEALTHY` | Fully operational |
| `WARNING` | Single pressure condition or approaching max age (>80%) |
| `ERROR` | Not ready, or 2+ pressure conditions |
| `TERMINAL` | Not ready AND unschedulable (kubelet down/unreachable) |

### Health evaluation logic

1. Evaluates 4 conditions: `Ready`, `DiskPressure`, `MemoryPressure`, `PIDPressure`
2. Aggregates to overall state:
   - **TERMINAL**: Not ready AND unschedulable
   - **ERROR**: Not ready, OR 2+ pressure conditions
   - **WARNING**: Single pressure, OR age > 80% of `max_age_hours`
   - **HEALTHY**: None of the above

### Pressure conditions

`DiskPressure`, `MemoryPressure`, `PIDPressure` — `True` means problem, `False` means nominal.

## MultipassProvider

Manages K3s node VMs through Multipass CLI. Infrastructure layer of the 48-hour lifecycle.

```python
from rubix_lite.k3s.provider import MultipassProvider

provider = MultipassProvider(
    cpus="2",           # default
    memory="2G",        # default
    disk="20G",         # default
    image="22.04",      # Ubuntu LTS, default
)
```

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `list_vms()` | `list[VMInfo]` | List all Multipass VMs |
| `get_vm(name)` | `VMInfo \| None` | Get single VM info |
| `create_vm(name, cloud_init=None)` | `VMInfo \| None` | Create new VM |
| `destroy_vm(name, purge=True)` | `bool` | Destroy VM and optionally purge |
| `join_cluster(vm_name, server_url, token)` | `bool` | Install K3s agent and join cluster |
| `replace_vm(old_name, server_url, token, cloud_init=None)` | `ReplaceResult` | Core 48-hour operation |

### Replace VM flow

```
1. Destroy old VM
2. Create fresh VM with same name
3. Join cluster (K3s agent install)
4. Return result with new IP and status
```

## Data Classes

All data classes are frozen (immutable).

### NodeInfo

```python
from rubix_lite.k3s.node import NodeInfo

node.name: str
node.creation_timestamp: datetime
node.age_hours: float
node.ready: bool
node.schedulable: bool
node.conditions: dict        # Kubernetes conditions
node.labels: dict            # Node labels
node.marked_for_termination: bool   # Property
node.terminate_reason: str | None   # Property
```

### DrainResult

```python
from rubix_lite.k3s.node import DrainResult

result.node_name: str
result.success: bool
result.pods_evicted: int
result.pods_failed: int
result.error: str | None
```

### NodeHealthReport

```python
from rubix_lite.k3s.health import NodeHealthReport, NodeCondition

report.node_name: str
report.state: HealthState
report.conditions: list[NodeCondition]
report.age_hours: float
report.max_age_hours: float
report.age_exceeded: bool
report.needs_termination: bool  # Property: aged out or TERMINAL
```

### NodeCondition

```python
condition.condition_type: str
condition.status: str
condition.healthy: bool
condition.message: str
```

### VMInfo

```python
from rubix_lite.k3s.provider import VMInfo

vm.name: str
vm.state: str
vm.ipv4: list[str]
vm.cpus: str
vm.memory: str
vm.disk: str
```

### ReplaceResult

```python
from rubix_lite.k3s.provider import ReplaceResult

result.old_vm: str
result.new_vm: str
result.success: bool
result.new_ip: str | None
result.error: str | None
```

## NodeTerminationPolicy CRD

Custom resource definition: `rubix.dev/v1alpha1` `NodeTerminationPolicy`

```yaml
apiVersion: rubix.dev/v1alpha1
kind: NodeTerminationPolicy
metadata:
  name: max-age-48h
spec:
  reason: "Node exceeded 48-hour maximum age"
  priority: 10            # 0-1000, higher = evaluated first
  enabled: true
  dryRun: false           # Label but don't drain
  selector:
    maxAgeHours: 48       # Match nodes older than N hours
    conditions:           # Match condition type/status
      - type: DiskPressure
        status: "True"
    nodeSelector: {}      # Label selector for node pools
  drain:
    timeoutSeconds: 300   # 30-3600
    deleteEmptydirData: true
```

## Default Policies

| Policy | Priority | Trigger | Drain Timeout |
|--------|----------|---------|--------------|
| `max-age-48h` | 10 | `maxAgeHours: 48` | 300s |
| `disk-pressure` | 100 | `DiskPressure == True` | 120s |
| `memory-pressure` | 100 | `MemoryPressure == True` | 120s |
| `not-ready` | 200 | `Ready == False` | 60s |

Higher priority policies are evaluated first. Health-based policies (100+) take precedence over age-based (10).
