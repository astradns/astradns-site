# Referencia de Valores de Helm

Referencia completa para `deploy/helm/astradns/values.yaml`.

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

### Sobreescrituras de Imagen del Motor

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

!!! warning "Se requiere cert-manager"
    El webhook requiere cert-manager para provisionar certificados TLS. Asegúrese de que cert-manager esté instalado y que el issuer exista antes de habilitar.

## Observabilidad

```yaml
serviceMonitor:
  enabled: false                     # Create ServiceMonitor for Prometheus Operator

grafana:
  dashboards:
    enabled: false                   # Create ConfigMap for Grafana sidecar discovery
```

## Integración con CoreDNS

```yaml
coredns:
  integration:
    enabled: false                   # Patch CoreDNS to forward external queries
```

Cuando se habilita, un Job de Kubernetes post-instalación:

1. Respalda el ConfigMap actual de CoreDNS
2. Agrega una directiva `forward` apuntando las zonas externas al agent de AstraDNS
3. Reinicia el deployment de CoreDNS para aplicar el cambio

## Perfil de Producción

Se proporciona un archivo de valores listo para producción en `values-production.yaml`:

```bash
helm install astradns deploy/helm/astradns \
  -f deploy/helm/astradns/values-production.yaml \
  --set webhook.certManager.issuerRef.name=your-issuer
```

Esto habilita: modo de red linkLocal, integración con CoreDNS, imposición por webhook, ServiceMonitor y dashboards de Grafana.
