# Configuración de Helm

## Instalación

### Desde el Chart OCI Oficial

```bash
helm upgrade --install astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system \
  --create-namespace
```

### Selección del Motor DNS

La unica eleccion de motor que el usuario necesita hacer es `agent.engineType`.

```bash
helm upgrade --install astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system \
  --create-namespace \
  --set agent.engineType=unbound
```

Valores soportados: `unbound`, `coredns`, `powerdns`, `bind`.

!!! info "Politica de imagenes"
    El chart fija automaticamente las imagenes oficiales:
    - `ghcr.io/astradns/astradns-operator:v<appVersion>`
    - `ghcr.io/astradns/astradns-agent:v<appVersion>-<engine>`

### Con Valores Personalizados

```bash
helm upgrade --install astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system \
  --create-namespace \
  -f my-values.yaml
```

## Perfiles de Topologia

### Node-Local (predeterminado)

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

## Perfiles Comunes

### Baseline de Produccion

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

### Cluster de Alto Trafico (Central)

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

## Validacion Post-Instalacion

```bash
# 1) Operator y agent saludables
kubectl -n astradns-system get pods -l app.kubernetes.io/component=operator
kubectl -n astradns-system get pods -l app.kubernetes.io/component=agent

# 2) ConfigMap existe
kubectl -n astradns-system get configmap astradns-agent-config

# 3) Smoke test DNS
kubectl run dns-test --rm -it --restart=Never --image=busybox:1.37 -- nslookup example.com
```

## Actualizacion

```bash
helm upgrade astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system \
  -f my-values.yaml
```

!!! tip "Ejecución en seco primero"
    Siempre previsualice los cambios antes de aplicar:
    ```bash
    helm upgrade astradns oci://ghcr.io/astradns/helm-charts/astradns \
      --namespace astradns-system \
      -f my-values.yaml \
      --dry-run --debug
    ```

## Renderizado de Plantillas

Para inspeccionar los manifiestos renderizados sin instalar:

```bash
helm template astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system \
  -f my-values.yaml
```

Consulte la [Referencia de Valores de Helm](../reference/helm-values.md) para todos los valores disponibles.
