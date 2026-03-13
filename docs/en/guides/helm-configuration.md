# Helm Configuration

## Installation

### Install from Local Chart

```bash
helm upgrade --install astradns deploy/helm/astradns \
  --namespace astradns-system \
  --create-namespace
```

### Choose the DNS Engine

The only engine-level choice required from users is `agent.engineType`.

```bash
helm upgrade --install astradns deploy/helm/astradns \
  --namespace astradns-system \
  --create-namespace \
  --set agent.engineType=unbound
```

Supported values: `unbound`, `coredns`, `powerdns`, `bind`.

!!! info "Image policy"
    The chart pins official images automatically:
    - `ghcr.io/astradns/astradns-operator:v<appVersion>`
    - `ghcr.io/astradns/astradns-agent:v<appVersion>-<engine>`

### Install with Custom Values

```bash
helm upgrade --install astradns deploy/helm/astradns \
  --namespace astradns-system \
  --create-namespace \
  -f my-values.yaml
```

## Topology Profiles

### Node-Local (default)

```yaml
agent:
  topology:
    profile: node-local
  engineType: unbound
  network:
    mode: linkLocal

clusterDNS:
  forwardExternalToAstraDNS:
    enabled: true
```

### Central

```yaml
agent:
  topology:
    profile: central
  engineType: unbound
  deployment:
    replicas: 3
  dnsService:
    type: ClusterIP
    sessionAffinity: ClientIP
```

## Common Profiles

### Production Baseline

```yaml
operator:
  leaderElect: true

agent:
  engineType: unbound
  topology:
    profile: node-local
  network:
    mode: linkLocal
  logMode: sampled
  logSampleRate: "0.1"

clusterDNS:
  forwardExternalToAstraDNS:
    enabled: true

webhook:
  enabled: true

serviceMonitor:
  enabled: true

grafana:
  dashboards:
    enabled: true
```

### High-Traffic Cluster (Central)

```yaml
agent:
  engineType: unbound
  topology:
    profile: central
  deployment:
    replicas: 5
  resources:
    requests:
      cpu: 250m
      memory: 256Mi
    limits:
      cpu: "1"
      memory: "1Gi"
  logMode: sampled
  logSampleRate: "0.05"
```

## Validate After Install

```bash
# 1) Operator and agent are healthy
kubectl -n astradns-system get pods -l app.kubernetes.io/component=operator
kubectl -n astradns-system get pods -l app.kubernetes.io/component=agent

# 2) ConfigMap exists
kubectl -n astradns-system get configmap astradns-agent-config

# 3) DNS smoke test
kubectl run dns-test --rm -it --restart=Never --image=busybox:1.37 -- nslookup example.com
```

## Upgrade

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

## Render Templates

To inspect the rendered manifests without installing:

```bash
helm template astradns deploy/helm/astradns \
  --namespace astradns-system \
  -f my-values.yaml
```

See the [Helm Values Reference](../reference/helm-values.md) for the full operational knobs.
