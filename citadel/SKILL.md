---
name: citadel
description: Lightweight K3s platform with zero-trust networking. Use when managing K3s clusters with Cilium CNI, 48-hour ephemeral nodes, NodeManager (cordon/drain), HealthChecker (4 states), MultipassProvider for VMs, or NodeTerminationPolicy CRDs. Triggers on citadel, rubix, k3s, cilium, ephemeral nodes, node lifecycle, zero-trust, eBPF, multipass.
---

# Citadel — K3s Platform (v0.1.0)

## Quick Start

```bash
pip install rubix-lite
```

```bash
./rubix cluster create my-cluster
./rubix network install        # Cilium CNI + Hubble
./rubix network policies       # Zero-trust default-deny
```

## Key Patterns

### 48-hour ephemeral node lifecycle
```
Create VM -> Join K3s -> Healthy (48h max) -> Mark for termination
-> Cordon -> Drain (Eviction API) -> Destroy VM -> Replace
```

### Python API
```python
from rubix_lite.k3s.node import NodeManager
from rubix_lite.k3s.health import HealthChecker, HealthState

manager = NodeManager()
checker = HealthChecker(max_age_hours=48)

for node in manager.list_nodes():
    report = checker.check_node(node)
    if report.needs_termination:
        manager.cordon(node.name)
        result = manager.drain(node.name, timeout=300)
```

## References

- **[api.md](references/api.md)** — NodeManager, HealthChecker, MultipassProvider, NodeTerminationPolicy CRD
- **[operations.md](references/operations.md)** — CLI reference, network policies, observability, security features
- **[examples.md](references/examples.md)** — Cluster setup, Apollo integration, node lifecycle, gotchas

## Grep Patterns

- `NodeManager|NodeInfo` — Find node management
- `HealthChecker|HealthState` — Find health monitoring
- `MultipassProvider|create_vm|destroy_vm` — Find VM lifecycle
- `rubix cluster|rubix network` — Find CLI usage
