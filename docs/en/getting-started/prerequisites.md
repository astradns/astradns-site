# Prerequisites

## Cluster Requirements

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| Kubernetes | v1.30+ | v1.32+ |
| Helm | v3.12+ | v3.16+ |
| Container runtime | Any OCI-compliant | containerd |
| CNI | Any | Calico, Cilium, or Flannel |

## Optional Dependencies

| Component | Required For | Installation |
|-----------|-------------|-------------|
| cert-manager | Validating webhook TLS | [cert-manager.io](https://cert-manager.io/docs/installation/) |
| Prometheus Operator | ServiceMonitor scraping | [prometheus-operator](https://github.com/prometheus-operator/prometheus-operator) |
| Grafana | DNS dashboards | Any Grafana with sidecar provisioning |

## Resource Requirements

### Operator (Deployment)

The operator is lightweight — it watches CRDs and writes ConfigMaps.

| Resource | Request | Limit |
|----------|---------|-------|
| CPU | 100m | 200m |
| Memory | 128Mi | 256Mi |

### Agent (DaemonSet)

The agent runs on every node and handles DNS traffic.

| Resource | Request | Limit |
|----------|---------|-------|
| CPU | 100m | 500m |
| Memory | 128Mi | 512Mi |

!!! tip "Sizing for high-traffic nodes"
    Nodes serving >5,000 QPS may need higher CPU limits. Monitor `astradns_queries_total` rate to identify hot nodes.

## Network Requirements

### hostPort Mode (default)

The agent binds to port `5353` on each node via `hostPort`. No special network configuration needed.

### linkLocal Mode (recommended for production)

The agent binds to the link-local address `169.254.20.11:53` on each node. This mode requires:

- `hostNetwork: true` on the agent DaemonSet (configured automatically by Helm)
- CoreDNS configured to forward external queries to `169.254.20.11` (automated via `clusterDNS.forwardExternalToAstraDNS.enabled=true`)

## RBAC

AstraDNS creates the following cluster-scoped RBAC resources:

- **ClusterRole** for the operator (read CRDs, write ConfigMaps, create events)
- **ClusterRoleBinding** binding the operator ServiceAccount
- **ServiceAccount** per component (operator, agent)

No cluster-admin privileges are required.
