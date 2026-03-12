# Helm Configuration

## Installation

### From Local Chart

```bash
helm install astradns deploy/helm/astradns \
  --namespace astradns-system \
  --create-namespace
```

### With Custom Values

```bash
helm install astradns deploy/helm/astradns \
  --namespace astradns-system \
  --create-namespace \
  -f my-values.yaml
```

## Common Configurations

### Minimal (Development)

```yaml
operator:
  replicas: 1
agent:
  engineType: unbound
  network:
    mode: hostPort
crds:
  install: true
```

### Production

```yaml
operator:
  replicas: 1
  leaderElect: true
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi

agent:
  engineType: unbound
  network:
    mode: linkLocal
  logMode: sampled
  logSampleRate: "0.1"
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

crds:
  install: true

webhook:
  enabled: true
  certManager:
    issuerRef:
      name: letsencrypt-prod
      kind: ClusterIssuer

serviceMonitor:
  enabled: true

grafana:
  dashboards:
    enabled: true

coredns:
  integration:
    enabled: true
```

### Air-Gapped Environment

```yaml
operator:
  image:
    repository: my-registry.internal/astradns/operator
    tag: "0.1.0"

agent:
  image:
    repository: my-registry.internal/astradns/agent
    tag: "0.1.0"
  engineImages:
    unbound: my-registry.internal/astradns/unbound:1.22

imagePullSecrets:
  - name: registry-credentials
```

### High-Traffic Nodes

```yaml
agent:
  resources:
    requests:
      cpu: 250m
      memory: 256Mi
    limits:
      cpu: "1"
      memory: "1Gi"
  logMode: errors-only   # Reduce log volume
```

## Upgrading

```bash
helm upgrade astradns deploy/helm/astradns \
  --namespace astradns-system \
  -f my-values.yaml
```

!!! tip "Dry run first"
    Always preview changes before applying:
    ```bash
    helm upgrade astradns deploy/helm/astradns \
      --namespace astradns-system \
      -f my-values.yaml \
      --dry-run --debug
    ```

## Templating

To inspect the rendered manifests without installing:

```bash
helm template astradns deploy/helm/astradns \
  --namespace astradns-system \
  -f my-values.yaml
```

See the [Helm Values Reference](../reference/helm-values.md) for all available values.
