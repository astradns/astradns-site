# Configuracao do Helm

## Instalacao

### A Partir do Chart Local

```bash
helm install astradns deploy/helm/astradns \
  --namespace astradns-system \
  --create-namespace
```

### Com Valores Personalizados

```bash
helm install astradns deploy/helm/astradns \
  --namespace astradns-system \
  --create-namespace \
  -f my-values.yaml
```

## Configuracoes Comuns

### Minima (Desenvolvimento)

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

### Producao

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

### Ambiente Air-Gapped

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

### Nos de Alto Trafego

```yaml
agent:
  resources:
    requests:
      cpu: 250m
      memory: 256Mi
    limits:
      cpu: "1"
      memory: "1Gi"
  logMode: errors-only   # Reduzir volume de logs
```

## Atualizacao

```bash
helm upgrade astradns deploy/helm/astradns \
  --namespace astradns-system \
  -f my-values.yaml
```

!!! tip "Execute um dry run primeiro"
    Sempre visualize as alteracoes antes de aplica-las:
    ```bash
    helm upgrade astradns deploy/helm/astradns \
      --namespace astradns-system \
      -f my-values.yaml \
      --dry-run --debug
    ```

## Renderizacao de Templates

Para inspecionar os manifests renderizados sem instalar:

```bash
helm template astradns deploy/helm/astradns \
  --namespace astradns-system \
  -f my-values.yaml
```

Consulte a [Referencia de Valores do Helm](../reference/helm-values.md) para todos os valores disponiveis.
