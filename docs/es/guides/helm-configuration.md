# Configuración de Helm

## Instalación

### Desde el Chart Local

```bash
helm install astradns deploy/helm/astradns \
  --namespace astradns-system \
  --create-namespace
```

### Con Valores Personalizados

```bash
helm install astradns deploy/helm/astradns \
  --namespace astradns-system \
  --create-namespace \
  -f my-values.yaml
```

## Configuraciones Comunes

### Mínima (Desarrollo)

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

### Producción

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

### Entorno Air-Gapped

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

### Nodos de Alto Tráfico

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

## Actualización

```bash
helm upgrade astradns deploy/helm/astradns \
  --namespace astradns-system \
  -f my-values.yaml
```

!!! tip "Ejecución en seco primero"
    Siempre previsualice los cambios antes de aplicar:
    ```bash
    helm upgrade astradns deploy/helm/astradns \
      --namespace astradns-system \
      -f my-values.yaml \
      --dry-run --debug
    ```

## Renderizado de Plantillas

Para inspeccionar los manifiestos renderizados sin instalar:

```bash
helm template astradns deploy/helm/astradns \
  --namespace astradns-system \
  -f my-values.yaml
```

Consulte la [Referencia de Valores de Helm](../reference/helm-values.md) para todos los valores disponibles.
