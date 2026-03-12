# Helm Values Reference

Complete reference for `deploy/helm/astradns/values.yaml`.

## Operator

```yaml
operator:
  image:
    repository: astradns/operator    # Container image repository
    tag: ""                          # Tag (defaults to Chart appVersion)
    pullPolicy: IfNotPresent
  replicas: 1                        # Number of operator replicas
  leaderElect: true                  # Enable leader election for HA
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi
  nodeSelector: {}
  tolerations: []
  affinity: {}
```

## Agent

```yaml
agent:
  image:
    repository: astradns/agent       # Container image repository
    tag: ""                          # Tag (defaults to Chart appVersion)
    pullPolicy: IfNotPresent
  engineType: unbound                # DNS engine: unbound, coredns, powerdns
  network:
    mode: hostPort                   # hostPort or linkLocal
    linkLocalIP: 169.254.20.11       # Link-local IP (when mode=linkLocal)
  logMode: sampled                   # full, sampled, errors-only, off
  logSampleRate: "0.1"               # Sample rate for sampled mode
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
  engineImages: {}                   # Per-engine image overrides
  nodeSelector: {}
  tolerations: []
  affinity: {}
```

### Engine Image Overrides

```yaml
agent:
  engineImages:
    coredns: "my-registry/coredns:1.12.1"
    powerdns: "my-registry/pdns-recursor:5.2"
```

## CRDs

```yaml
crds:
  install: true                      # Install CRDs with the chart
```

## Webhook

```yaml
webhook:
  enabled: false                     # Enable validating webhook
  certManager:
    issuerRef:
      name: ""                       # cert-manager Issuer name (required when enabled)
      kind: ClusterIssuer            # Issuer or ClusterIssuer
```

!!! warning "cert-manager required"
    The webhook requires cert-manager to provision TLS certificates. Ensure cert-manager is installed and the issuer exists before enabling.

## Observability

```yaml
serviceMonitor:
  enabled: false                     # Create ServiceMonitor for Prometheus Operator

grafana:
  dashboards:
    enabled: false                   # Create ConfigMap for Grafana sidecar discovery
```

## CoreDNS Integration

```yaml
coredns:
  integration:
    enabled: false                   # Patch CoreDNS to forward external queries
```

When enabled, a post-install Kubernetes Job:

1. Backs up the current CoreDNS ConfigMap
2. Adds a `forward` directive pointing external zones to the AstraDNS agent
3. Restarts the CoreDNS deployment to pick up the change

## Production Profile

A production-ready values file is provided at `values-production.yaml`:

```bash
helm install astradns deploy/helm/astradns \
  -f deploy/helm/astradns/values-production.yaml \
  --set webhook.certManager.issuerRef.name=your-issuer
```

This enables: linkLocal network mode, CoreDNS integration, webhook enforcement, ServiceMonitor, and Grafana dashboards.
