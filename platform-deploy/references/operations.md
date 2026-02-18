# Platform Deploy — Operations & Monitoring

> Part of the platform-deploy workflow. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [Entity Monitoring](#entity-monitoring)
- [Health Check Integration](#health-check-integration)
- [Citadel Node Lifecycle](#citadel-node-lifecycle)
- [Network Policies](#network-policies)
- [Observability Stack](#observability-stack)
- [Troubleshooting](#troubleshooting)
- [Gotchas](#gotchas)

## Entity Monitoring

### Check entity status

```bash
# Single entity
apollo entity status ENTITY_ID

# All entities in an environment
apollo entity list --environment production

# Pending plans requiring attention
apollo plan list --state pending

# Output as JSON for scripting
apollo entity list --environment production --format json
```

### Entity health events

Subscribe to entity state changes for alerting:

```python
from apollo.events import entity_state_changed, plan_failed, plan_completed

@entity_state_changed.connect
def on_state_change(sender, entity_id, old_state, new_state, **kwargs):
    if new_state == "failed":
        pagerduty.alert(f"Entity {entity_id} entered FAILED state")
    elif new_state == "degraded":
        slack.post(f"Entity {entity_id} degraded — health checks failing")

@plan_failed.connect
def on_plan_failed(sender, plan_id, error, **kwargs):
    slack.post(f"Plan {plan_id} failed: {error}")

@plan_completed.connect
def on_plan_completed(sender, plan_id, **kwargs):
    slack.post(f"Plan {plan_id} completed successfully")
```

### Entity state recovery

| Current State | Recovery Action |
|--------------|-----------------|
| `DEGRADED` | Automatic — returns to `RUNNING` when health checks pass |
| `FAILED` | Orchestration engine proposes rollback plan automatically |
| `INSTALLING` (stuck) | Check spoke agent status: `apollo spoke status` |

## Health Check Integration

The service-launcher health check (configured via `check_args`, `check_command`, or `check_script` in the BUILD target) feeds into entity state transitions:

```
Container starts
  → service-launcher runs health check
    → K8s readiness probe succeeds/fails
      → Spoke agent reports state to Apollo Hub
        → Entity state: INSTALLING → RUNNING (success)
        → Entity state: INSTALLING → FAILED (failure)
        → Entity state: RUNNING → DEGRADED (health regression)
```

```bash
# Check the health check configuration in the distribution
cat /opt/services/<name>-<version>/service/run/launcher-check.yml

# Manual health check
/opt/services/<name>-<version>/service/monitoring/bin/check.sh
```

## Citadel Node Lifecycle

### 48-hour rotation

Citadel nodes are ephemeral — maximum 48-hour lifespan. The lifecycle controller runs continuously:

```
Create VM → Join K3s → Healthy (48h max) → Cordon → Drain → Destroy → Replace
```

```bash
# Check node ages and status
./rubix node status

# Output format:
# NODE              AGE            MAX AGE         STATUS
# my-node-1         2d 5h          48h             OK (19h left)
# my-node-2         1d 22h         48h             WARNING (2h left)
# my-node-3         2d 1h          48h             EXCEEDED (1h over)

# Manual drain (respects PodDisruptionBudgets)
./scripts/node-lifecycle.sh drain <node>

# Dry run — check what would happen
DRY_RUN=true ./scripts/node-lifecycle.sh cycle
```

### Node health states

The `HealthChecker` evaluates 4 Kubernetes conditions:

| State | Meaning |
|-------|---------|
| `HEALTHY` | Fully operational |
| `WARNING` | Single pressure condition or approaching max age (>80%) |
| `ERROR` | Not ready, or 2+ pressure conditions |
| `TERMINAL` | Not ready AND unschedulable (kubelet down) |

### Default termination policies

| Policy | Priority | Trigger | Drain Timeout |
|--------|----------|---------|--------------|
| `max-age-48h` | 10 | Node older than 48 hours | 300s |
| `disk-pressure` | 100 | `DiskPressure == True` | 120s |
| `memory-pressure` | 100 | `MemoryPressure == True` | 120s |
| `not-ready` | 200 | `Ready == False` | 60s |

Higher priority policies are evaluated first. Health-based policies (100+) take precedence over age-based (10).

### Impact on deployed entities

When a node is drained, pods are evicted via the Kubernetes Eviction API:
1. PodDisruptionBudgets are respected — if PDB prevents eviction, retry with backoff
2. Pods are rescheduled to healthy nodes by the K8s scheduler
3. Entity state may briefly transition to `DEGRADED` if replicas drop below minimum
4. Spoke agent continues reporting state; entity returns to `RUNNING` once pods reschedule

Set `replication_min >= 2` in the `sls_service` BUILD target to maintain availability during node drains.

## Network Policies

Citadel uses Cilium `CiliumNetworkPolicy` for identity-based (pod label) network control.

### Policy layers

```
Layer 1: Default deny all (foundation)
Layer 2: System namespace policies (CoreDNS, Cilium, API server)
Layer 3: Application policies (per-tier rules)
Layer 4: FQDN egress control (per-service external access)
```

### Zero-trust model

All traffic is denied by default. Each service must explicitly declare its network needs:

```
Frontend          Backend          Database
  ↓ HTTP (80)       ↓ HTTP (5678)    ↓ Redis (6379)
  from: world       from: frontend   from: backend ONLY
  to: backend       to: database     to: NOTHING
```

### FQDN egress control

Restrict external traffic by domain name:

| Policy | Allowed Domains |
|--------|----------------|
| `backend-fqdn-egress` | `api.github.com`, `api.stripe.com`, `*.amazonaws.com` |
| `database-no-egress` | Zero egress (fully isolated) |
| `internal-only-egress` | `*.svc.cluster.local`, internal domains only |

### Testing policies

```bash
./rubix network policies     # Apply all network policies
./rubix network test         # Run connectivity tests
hubble observe --verdict DROPPED  # See policy violations in real time
```

## Observability Stack

### Hubble (network flows)

```bash
hubble observe                           # Real-time traffic flows
hubble observe --verdict DROPPED         # Policy violations only
hubble observe --to-pod backend          # Traffic to specific pod
```

Hubble UI provides visual service dependency maps.

### Prometheus + Grafana

```bash
./rubix monitor install
```

Metrics collected:
- Node resource usage (CPU, memory, disk)
- Pod counts and states
- Network policy enforcement (allowed/denied flows)
- Cilium agent health
- Node ages and lifecycle events

### Apollo events

Apollo Hub emits events for all lifecycle transitions. Subscribe for monitoring:

| Signal | Fires when |
|--------|-----------|
| `entity_state_changed` | Entity FSM transition |
| `plan_failed` | Plan execution fails |
| `plan_completed` | Plan executes successfully |
| `release_promoted` | Release promoted to new channel |

### Security

```bash
./rubix security enable-wireguard   # Pod-to-pod encryption (eBPF-integrated)
./rubix security enable-audit       # K8s audit logging (FedRAMP/SOC2)
```

## Troubleshooting

### Entity stuck in INSTALLING

```bash
# 1. Check spoke agent connectivity
apollo spoke status

# 2. Check if plan is executing
apollo plan list --state executing

# 3. Check Helm release on cluster
helm list -n <namespace>
helm status <release-name>

# 4. Check pod events
kubectl describe pod -l app=<service-name>
```

### Entity in FAILED state

```bash
# 1. Check what failed
apollo entity status ENTITY_ID

# 2. Orchestration engine auto-proposes rollback
apollo plan list --state pending

# 3. Approve rollback
apollo plan approve PLAN_ID --by oncall@team.com

# 4. Or create manual rollback
apollo plan create --entity ENTITY_ID --type rollback --target-version 0.9.0
```

### Plan blocked by constraints

```bash
# 1. See why plan is blocked
apollo plan list --state blocked

# 2. Check maintenance windows
apollo maintenance list

# 3. Check suppression windows (holiday freezes)
# Suppression windows are absolute date ranges — wait or remove

# 4. Check dependency health
apollo entity list --environment production
```

### Node drain stuck

```bash
# 1. Check for PDB conflicts
kubectl get pdb -A

# 2. Check which pods are blocking
kubectl get pods --field-selector spec.nodeName=<node> -A

# 3. Drain with extended timeout
DRAIN_TIMEOUT=600 ./scripts/node-lifecycle.sh drain <node>
```

### Network policy blocking traffic

```bash
# 1. Check for DROPPED verdicts
hubble observe --verdict DROPPED --to-pod <pod-name>

# 2. Verify policy is applied
kubectl get cnp -A   # CiliumNetworkPolicy

# 3. Test connectivity
./rubix network test
```

## Gotchas

1. **48-hour nodes are non-negotiable**: Citadel destroys nodes after 48 hours regardless of workload. Design services for graceful eviction with `replication_min >= 2`
2. **PDB can block node drains indefinitely**: If a PodDisruptionBudget prevents eviction and timeout is reached, the drain may fail. Monitor drain operations
3. **Default deny blocks everything**: New services have zero network access until explicit `CiliumNetworkPolicy` rules are added. Test connectivity before production
4. **Entity events are synchronous**: Blinker signals fire synchronously. Long-running event handlers (PagerDuty, Slack) block the caller. Use background workers
5. **Spoke agent needs Hub connectivity**: Offline mode queues commands in Redis but requires eventual network access. Check `apollo spoke status` if entities aren't progressing
6. **Maintenance windows are cron-based**: A `cron_expression` of `0 2 * * 6` means "2 AM Saturday" — not a duration. Pair with `duration_hours` to define the window
7. **Suppression windows block all changes**: Unlike maintenance windows (recurring), suppression windows are one-time date ranges that block everything. No override without removing the window
8. **Health-based termination overrides age**: A node with `DiskPressure` (priority 100) is terminated before an aged-out node (priority 10), even if the aged node is older
9. **Rollback is automatic on FAILED**: When an entity transitions to `FAILED`, the orchestration engine proposes a rollback plan. It still requires approval in production
10. **Version ordering is SLS-specific**: `1.0.0-rc1 < 1.0.0` in SLS ordering. Release candidates sort before releases, not after
