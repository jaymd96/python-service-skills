# Citadel — Examples & Gotchas

> Part of the citadel skill. See [SKILL.md](../SKILL.md) for overview.

## Cluster Setup

```bash
# Create a 3-node cluster
./rubix cluster create my-cluster --nodes 3

# Install Cilium CNI with Hubble
./rubix network install

# Apply zero-trust default-deny policies
./rubix network policies

# Install lifecycle CRDs
./rubix lifecycle install

# Verify
./rubix node status
./rubix lifecycle policies
```

## Node Health Monitoring

```python
from rubix_lite.k3s.node import NodeManager
from rubix_lite.k3s.health import HealthChecker, HealthState

manager = NodeManager()
checker = HealthChecker(max_age_hours=48)

for node in manager.list_nodes():
    report = checker.check(node)
    print(f"{node.name}: {report.state.name} "
          f"(age: {report.age_hours:.1f}h, "
          f"needs_termination: {report.needs_termination})")
```

## 48-Hour Node Rotation

```python
from rubix_lite.k3s.node import NodeManager
from rubix_lite.k3s.health import HealthChecker
from rubix_lite.k3s.provider import MultipassProvider

manager = NodeManager()
checker = HealthChecker(max_age_hours=48)
provider = MultipassProvider()

for node in manager.list_nodes():
    report = checker.check(node)
    if report.needs_termination:
        # Label for termination
        manager.label_for_termination(
            node.name,
            reason="Exceeded 48-hour age limit",
            policy_name="max-age-48h",
        )
        # Cordon and drain (respects PDBs)
        manager.cordon(node.name)
        result = manager.drain(node.name, timeout_seconds=300)
        if result.success:
            # Replace VM
            replace = provider.replace_vm(
                old_name=node.name,
                server_url="https://k3s-server:6443",
                token="node-token",
            )
            print(f"Replaced {node.name} -> new IP: {replace.new_ip}")
        else:
            print(f"Drain failed: {result.error}")
```

## Apollo Integration

Apollo deploys services to Citadel's K3s clusters:

```python
# Apollo SpokeAgent runs on Citadel cluster
from apollo.spoke.agent import SpokeAgent, SpokeAgentConfig

config = SpokeAgentConfig(
    hub_url="https://hub.apollo.internal",
    namespace="default",
    poll_interval=30,
)
agent = SpokeAgent(config)
agent.start()

# Citadel rotates nodes beneath Apollo's workloads
# PDB-aware drain ensures zero-downtime during rotation
```

## Custom NodeTerminationPolicy

```yaml
apiVersion: rubix.dev/v1alpha1
kind: NodeTerminationPolicy
metadata:
  name: custom-age-24h
  namespace: rubix-system
spec:
  reason: "Staging nodes rotated every 24 hours"
  priority: 10
  enabled: true
  dryRun: false
  selector:
    maxAgeHours: 24
    nodeSelector:
      matchLabels:
        env: staging
  drain:
    timeoutSeconds: 120
    deleteEmptydirData: true
```

## Health-Based Termination

```yaml
apiVersion: rubix.dev/v1alpha1
kind: NodeTerminationPolicy
metadata:
  name: disk-pressure-critical
  namespace: rubix-system
spec:
  reason: "Node under critical disk pressure"
  priority: 150
  enabled: true
  selector:
    conditions:
      - type: DiskPressure
        status: "True"
  drain:
    timeoutSeconds: 60
    deleteEmptydirData: true
```

## Network Policy Testing

```bash
# Apply policies
./rubix network policies

# Test connectivity
./rubix network test

# Observe traffic flows
hubble observe --verdict DROPPED     # See denied traffic
hubble observe --from-pod frontend   # Traffic from frontend
hubble observe --to-pod database     # Traffic to database

# Verify zero-trust
kubectl exec frontend -- curl backend:5678    # Should work
kubectl exec frontend -- curl database:6379   # Should be DENIED
kubectl exec backend -- curl database:6379    # Should work
```

## Dry Run Mode

Test policies without actually draining:

```yaml
apiVersion: rubix.dev/v1alpha1
kind: NodeTerminationPolicy
spec:
  dryRun: true    # Label nodes but don't drain
  # ...
```

```bash
# Also available in shell script
DRY_RUN=true ./scripts/node-lifecycle.sh cycle
```

## Custom HealthChecker Threshold

```python
# Shorter lifecycle for dev environments
checker = HealthChecker(max_age_hours=12)

# Longer lifecycle for stable production
checker = HealthChecker(max_age_hours=96)
```

## Gotchas

1. **48-hour lifecycle is the default, not a requirement**: `max_age_hours` is configurable per-policy and per-`HealthChecker` instance. Set it to match your security/compliance needs
2. **Drain uses Eviction API, not force delete**: The `drain()` method respects PodDisruptionBudgets. If a PDB prevents eviction, it retries with backoff. Set `timeout_seconds` appropriately
3. **HealthChecker is read-only**: It only evaluates and reports. It never cordons, drains, or destroys nodes. Combine it with `NodeManager` for action
4. **MultipassProvider is for development**: It manages local VMs via Multipass CLI. For production, replace with cloud provider APIs (EC2, GCE, etc.)
5. **Default deny blocks everything**: After applying `01-default-deny.yaml`, all traffic is blocked except DNS and API server. Apply application policies immediately or pods will be isolated
6. **Policy priority determines evaluation order**: Higher numbers evaluate first. Use 100+ for health-based (urgent), 10 for age-based (routine)
7. **CRD must be installed first**: Run `./rubix lifecycle install` before creating NodeTerminationPolicy resources. The CRD, RBAC, and ServiceAccount are all required
8. **`needs_termination` is a property, not a state**: `NodeHealthReport.needs_termination` returns `True` if the node is aged out OR in `TERMINAL` state. It doesn't consider `WARNING` or `ERROR` as termination triggers
9. **WireGuard encryption is transparent**: Once enabled via `./rubix security enable-wireguard`, all pod-to-pod traffic is encrypted. No application changes needed, but there's a small performance overhead
10. **K3s install URL is hardcoded**: `MultipassProvider` uses `https://get.k3s.io` for agent installation. For air-gapped environments, pre-install K3s on the VM image
