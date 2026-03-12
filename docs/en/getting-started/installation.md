# Installation

## Helm Install

### Basic Installation

```bash
helm install astradns deploy/helm/astradns \
  --namespace astradns-system \
  --create-namespace
```

### Production Installation

Use the production values profile for a hardened deployment:

```bash
helm install astradns deploy/helm/astradns \
  --namespace astradns-system \
  --create-namespace \
  -f deploy/helm/astradns/values-production.yaml \
  --set webhook.certManager.issuerRef.name=your-cluster-issuer
```

This enables:

- **Link-local data path** — agent binds to `169.254.20.11` for true node-local DNS
- **CoreDNS integration** — automatic patching of cluster DNS to forward external queries
- **Validating webhook** — enforces one `DNSUpstreamPool` per namespace
- **ServiceMonitor** — Prometheus scraping for operator and agent metrics
- **Grafana dashboard** — pre-built DNS observability dashboard

### Verify Installation

```bash
# All pods should be Running
kubectl get pods -n astradns-system

# CRDs should be registered
kubectl get crds | grep astradns

# Expected output:
# dnscacheprofiles.dns.astradns.com
# dnsupstreampools.dns.astradns.com
# externaldnspolicies.dns.astradns.com
```

## Post-Install Configuration

### Create an Upstream Pool

Every namespace that uses AstraDNS needs a `DNSUpstreamPool`:

```yaml
apiVersion: dns.astradns.com/v1alpha1
kind: DNSUpstreamPool
metadata:
  name: production
  namespace: astradns-system
spec:
  upstreams:
    - address: "1.1.1.1"
    - address: "8.8.8.8"
    - address: "9.9.9.9"
  healthCheck:
    enabled: true
    intervalSeconds: 30
    timeoutSeconds: 5
    failureThreshold: 3
  loadBalancing:
    strategy: round-robin
```

### Configure Caching (Optional)

```yaml
apiVersion: dns.astradns.com/v1alpha1
kind: DNSCacheProfile
metadata:
  name: default
  namespace: astradns-system
spec:
  maxEntries: 50000
  positiveTtl:
    minSeconds: 60
    maxSeconds: 300
  negativeTtl:
    seconds: 30
  prefetch:
    enabled: true
    threshold: 5
```

!!! note
    The cache profile named `default` is automatically picked up by the operator. No explicit reference is needed in the upstream pool.

## Uninstall

```bash
# Remove CRs first
kubectl delete dnsupstreampools,dnscacheprofiles,externaldnspolicies \
  --all -n astradns-system

# Uninstall the chart
helm uninstall astradns -n astradns-system

# Remove CRDs (optional)
kubectl delete crds \
  dnsupstreampools.dns.astradns.com \
  dnscacheprofiles.dns.astradns.com \
  externaldnspolicies.dns.astradns.com

# Remove namespace
kubectl delete namespace astradns-system
```

!!! warning
    Deleting CRDs will remove all AstraDNS custom resources across all namespaces. This is irreversible.
