# Instalación

## Instalación con Helm

### Instalación Básica

```bash
helm upgrade --install astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system \
  --create-namespace \
  --set agent.engineType=unbound
```

### Instalación para Producción

Use el perfil de valores de producción para un despliegue reforzado:

```bash
helm upgrade --install astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system \
  --create-namespace \
  -f values-production.yaml \
  --set webhook.certManager.issuerRef.name=your-cluster-issuer
```

Esto habilita:

- **Ruta de datos link-local** -- el agent se enlaza a `169.254.20.11` para DNS verdaderamente local al nodo
- **Integración con CoreDNS** -- parcheo automático del DNS del clúster para reenviar consultas externas
- **Webhook de validación** -- impone un solo `DNSUpstreamPool` por namespace
- **ServiceMonitor** -- scraping de Prometheus para métricas del operator y del agent
- **Dashboard de Grafana** -- dashboard de observabilidad DNS preconfigurado

### Verificar la Instalación

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

## Configuración Post-Instalación

### Crear un Pool de Upstreams

Cada namespace que usa AstraDNS necesita un `DNSUpstreamPool`:

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

### Configurar Caché (Opcional)

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
    El perfil de caché llamado `default` es tomado automáticamente por el operator. No se necesita una referencia explícita en el pool de upstreams.

## Desinstalación

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
    Eliminar los CRDs removerá todos los recursos personalizados de AstraDNS en todos los namespaces. Esta acción es irreversible.
