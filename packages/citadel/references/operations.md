# Citadel — Operations & Security

> Part of the citadel skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [CLI Reference](#cli-reference)
- [Network Policies](#network-policies)
- [Zero-Trust Model](#zero-trust-model)
- [FQDN Egress Control](#fqdn-egress-control)
- [Cilium Installation](#cilium-installation)
- [Observability](#observability)
- [Security Features](#security-features)
- [RBAC Configuration](#rbac-configuration)

## CLI Reference

```bash
# Cluster management
./rubix cluster create <name> [--nodes N]    # Create K3s cluster with Multipass VMs
./rubix cluster delete <name>                # Delete all cluster VMs
./rubix cluster list                         # List all VMs

# Node management
./rubix node status                          # Show node ages and health
./rubix node list                            # List nodes with lifecycle info

# Lifecycle
./rubix lifecycle install                    # Install CRDs and RBAC
./rubix lifecycle policies                   # Show active NodeTerminationPolicies

# Network
./rubix network install                      # Install Cilium CNI
./rubix network policies                     # Apply network policies
./rubix network test                         # Test network policies

# Monitoring
./rubix monitor install                      # Install Prometheus + Grafana

# Security
./rubix security enable-wireguard            # Enable WireGuard encryption
./rubix security enable-audit                # Enable audit logging
```

### Node lifecycle script

```bash
# Check node ages
./scripts/node-lifecycle.sh check [node]

# Drain a specific node
./scripts/node-lifecycle.sh drain <node>

# Check and drain all aged nodes
./scripts/node-lifecycle.sh cycle

# Show node age summary
./scripts/node-lifecycle.sh status

# Configuration
NODE_MAX_AGE_HOURS=48     # default
DRAIN_TIMEOUT=300         # default
DRY_RUN=true              # test mode
```

## Network Policies

Citadel uses Cilium `CiliumNetworkPolicy` for identity-based (pod labels), not IP-based, network control.

### Policy layers

```
Layer 1: Default deny all (foundation)
Layer 2: System namespace policies (CoreDNS, Cilium, API server)
Layer 3: Application policies (per-tier rules)
Layer 4: FQDN egress control (per-service external access)
```

### Default deny (`01-default-deny.yaml`)

```yaml
# Deny all ingress and egress by default
# Then allow:
- DNS to CoreDNS (UDP/TCP 53)
- API server access (TCP 6443)
- Kubelet health checks
```

### System policies (`02-system-policies.yaml`)

| Policy | Allows |
|--------|--------|
| `cilium-agent-policy` | Cilium agent communication |
| `hubble-relay-policy` | Hubble metrics collection |
| `hubble-ui-policy` | Hubble UI web access (TCP 8081) |
| `coredns-policy` | DNS resolution (TCP/UDP 53) |

## Zero-Trust Model

3-tier application example (`03-demo-app-policies.yaml`):

```
Frontend          Backend          Database
  ↓ HTTP (80)       ↓ HTTP (5678)    ↓ Redis (6379)
  from: world       from: frontend   from: backend ONLY
  to: backend       to: database     to: NOTHING
```

| Tier | Ingress | Egress |
|------|---------|--------|
| Frontend | HTTP (80) from world | To backend (5678) |
| Backend | From frontend (5678) | To database (6379) |
| Database | From backend (6379) ONLY | No egress |
| Test client | Access all demo services | For testing |

### L7 support

Cilium supports L7 (HTTP method/path) policies in addition to L3/L4 (port/protocol):

```yaml
# Allow only GET /api/* to backend
rules:
  http:
    - method: GET
      path: "/api/.*"
```

## FQDN Egress Control

Restrict external traffic by domain name (`05-fqdn-egress-policies.yaml`):

| Policy | Allowed Domains |
|--------|----------------|
| `backend-fqdn-egress` | `api.github.com`, `api.stripe.com`, `*.amazonaws.com`, `*.storage.googleapis.com` |
| `compute-fqdn-egress` | `*.s3.amazonaws.com`, PyPI, data sources |
| `database-no-egress` | Zero egress (fully isolated) |
| `internal-only-egress` | `*.svc.cluster.local`, internal domains only |

## Cilium Installation

Installed via Helm with full observability:

```bash
./rubix network install
```

### Key Helm values

| Setting | Value | Purpose |
|---------|-------|---------|
| `kubeProxyReplacement` | `true` | eBPF-based load balancing |
| `hubble.enabled` | `true` | Network observability |
| `hubble.relay.enabled` | `true` | Cluster-wide visibility |
| `hubble.ui.enabled` | `true` | Visual service maps |
| `prometheus.enabled` | `true` | Metrics collection |

### Installed tools

- **Cilium CLI**: `cilium status`, `cilium connectivity test`
- **Hubble CLI**: `hubble observe`, flow inspection

## Observability

### Hubble

Network flow observability:
- `hubble observe` — real-time traffic flows
- `hubble observe --verdict DROPPED` — policy violations
- Hubble UI for visual service dependency maps

### Prometheus + Grafana

```bash
./rubix monitor install
```

Metrics collected:
- Node resource usage (CPU, memory, disk)
- Pod counts and states
- Network policy enforcement (allowed/denied flows)
- Cilium agent health

### Node status format

```
NODE              AGE            MAX AGE         STATUS
----              ---            -------         ------
my-node-1         2d 5h          48h             OK (19h left)
my-node-2         1d 22h         48h             WARNING (2h left)
my-node-3         2d 1h          48h             EXCEEDED (1h over)
```

## Security Features

### WireGuard encryption

```bash
./rubix security enable-wireguard
```

Transparent pod-to-pod encryption using WireGuard (eBPF-integrated through Cilium). No application changes required.

### Audit logging

```bash
./rubix security enable-audit
```

Kubernetes audit logging for compliance (FedRAMP/SOC2).

### PodDisruptionBudget awareness

The drain operation uses the Kubernetes Eviction API, which respects PDBs:
- If PDB prevents eviction → retry with exponential backoff (429 status)
- Graceful pod termination with grace period
- No force-delete unless timeout exceeded

## RBAC Configuration

**ServiceAccount**: `rubix-lifecycle-controller` in `rubix-system` namespace.

### ClusterRole permissions

| Resource | Verbs |
|----------|-------|
| `nodes` | get, list, watch, patch, update |
| `nodes/status` | get |
| `pods/eviction` | create |
| `pods` | get, list, watch |
| `nodeterminationpolicies` | full CRUD |
| `nodeterminationpolicies/status` | get, update, patch |
| `events` | create, patch |
| `poddisruptionbudgets` | get, list, watch |
| `leases` | full CRUD (leader election) |
```
