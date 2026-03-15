# Getting Started

AstraDNS installs as a Helm chart and requires no changes to your application code. This guide walks you through a minimal setup on any Kubernetes cluster.

## What You Get

After installation, every node in your cluster runs an AstraDNS Agent that:

- **Proxies** external DNS queries through a managed resolver (Unbound by default)
- **Caches** responses with configurable TTLs
- **Logs** every query as structured JSON
- **Exports** Prometheus metrics on cache hits, latency, failures, and upstream health
- **Health-checks** upstream resolvers and reports status via CRD conditions

## 5-Minute Install

### 1. Add the Helm chart

```bash
helm upgrade --install astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system \
  --create-namespace \
  --set agent.engineType=unbound \
  --set agent.network.mode=linkLocal \
  --set clusterDNS.forwardExternalToAstraDNS.enabled=true
```

### 2. Create an upstream pool

```yaml
apiVersion: dns.astradns.com/v1alpha1
kind: DNSUpstreamPool
metadata:
  name: default
  namespace: astradns-system
spec:
  upstreams:
    - address: "1.1.1.1"
      port: 53
    - address: "8.8.8.8"
      port: 53
```

```bash
kubectl apply -f pool.yaml
```

### 3. Verify

```bash
# Check operator is running
kubectl get pods -n astradns-system -l app.kubernetes.io/component=operator

# Check agent is running on nodes
kubectl get pods -n astradns-system -l app.kubernetes.io/component=agent

# Check the pool status
kubectl get dnsupstreampools -n astradns-system
```

The pool should show `Ready=True` within seconds.

### 4. Test DNS resolution

```bash
kubectl run dns-test --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup example.com
```

## Next Steps

- [Prerequisites](prerequisites.md) for production clusters
- [Installation](installation.md) with production values
- [Helm Configuration](../guides/helm-configuration.md) reference
- [CoreDNS Integration](../guides/coredns-integration.md) for node-local DNS
