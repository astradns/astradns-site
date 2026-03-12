# Guía de Actualización

## Proceso de Actualización

### 1. Revisar las Notas de la Versión

Antes de actualizar, revise las notas de la versión en busca de cambios incompatibles, nuevas funcionalidades y deprecaciones.

### 2. Respaldo

```bash
# Backup CRDs
kubectl get dnsupstreampools,dnscacheprofiles,externaldnspolicies \
  -n astradns-system -o yaml > astradns-backup.yaml

# Backup Helm values
helm get values astradns -n astradns-system > current-values.yaml

# Note current revision
helm history astradns -n astradns-system
```

### 3. Ejecución en Seco

```bash
helm upgrade astradns deploy/helm/astradns \
  --namespace astradns-system \
  -f current-values.yaml \
  --dry-run --debug
```

Revise la salida en busca de cambios inesperados.

### 4. Actualizar

```bash
helm upgrade astradns deploy/helm/astradns \
  --namespace astradns-system \
  -f current-values.yaml
```

### 5. Verificar

```bash
# Check pods are running
kubectl get pods -n astradns-system

# Verify CRD status
kubectl get dnsupstreampools -n astradns-system

# Test DNS resolution
kubectl run dns-test --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup example.com

# Check metrics
kubectl port-forward -n astradns-system ds/astradns-agent 9153:9153 &
curl -s http://localhost:9153/metrics | grep astradns_queries_total
```

## Reversión

Si la actualización causa problemas:

```bash
helm rollback astradns <previous-revision> -n astradns-system
```

## Actualizaciones de CRDs

!!! warning "Los cambios de CRDs requieren atención manual"
    Helm no actualiza los CRDs con `helm upgrade`. Si la nueva versión incluye cambios en los CRDs, aplíquelos manualmente:

    ```bash
    kubectl apply -f deploy/helm/astradns/crds/
    ```

## Notas Específicas por Versión

### v0.1.x

Versión inicial. Sin cambios incompatibles dentro de la serie v0.1.x.

**Limitaciones conocidas:**

- Las métricas a nivel de namespace aparecen como "unknown" (requiere la ruta de datos directa en una versión futura)
- ExternalDNSPolicy valida referencias pero no impone el enrutamiento
- Solo un DNSUpstreamPool por namespace (por diseño; el webhook lo impone cuando está habilitado)
