# Resolución de Problemas

## Diagnóstico Rápido

```bash
# One-liner health check
kubectl get pods,dnsupstreampools,dnscacheprofiles -n astradns-system
```

## Problemas Comunes

### "No CRDs to install; skipping" durante helm install

**Causa:** Los CRDs no se están instalando.

**Solución:** Asegúrese de que `crds.install=true` en sus valores:

```bash
helm upgrade --install astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system --create-namespace \
  --set crds.install=true
```

### El pod del agent está en CrashLoopBackOff

**Diagnóstico:**

```bash
kubectl logs -n astradns-system -l app.kubernetes.io/component=agent --previous
kubectl describe pod -n astradns-system -l app.kubernetes.io/component=agent
```

**Causas comunes:**

| Mensaje de log | Causa | Solución |
|----------------|-------|----------|
| `exec: "unbound": not found` | Tipo de motor incorrecto para la imagen | Haga coincidir `agent.engineType` con el binario del motor en la imagen |
| `listen tcp :5353: bind: address already in use` | Conflicto de puerto | Verifique si hay otros servicios en el puerto 5353 |
| `open /etc/astradns/config/config.json: no such file` | ConfigMap no montado | Verifique que el ConfigMap exista y esté referenciado en la carga del agent (DaemonSet/Deployment) |

### El pool muestra Ready=False, Reason=Superseded

**Causa:** Existen múltiples recursos `DNSUpstreamPool` en el namespace. Solo el pool más antiguo está activo.

**Solución:**

```bash
# See all pools and their status
kubectl get dnsupstreampools -n astradns-system -o yaml

# Delete the superseded pool
kubectl delete dnsupstreampool <superseded-name> -n astradns-system
```

### Las métricas muestran cero aciertos de caché

**Posibles causas:**

1. **Caché no calentado:** Ejecute consultas repetidas al mismo dominio y verifique nuevamente
2. **Caché deshabilitado:** Verifique que `DNSCacheProfile` exista y que `maxEntries > 0`
3. **Limitación del proxy:** El proxy reporta el estado del caché como "unknown" -- las métricas de caché provienen del motor, no del proxy

### Alta tasa de SERVFAIL

**Diagnóstico:**

```bash
# Check upstream health
curl -s http://localhost:9153/metrics | grep astradns_upstream_healthy

# Check upstream latency
curl -s http://localhost:9153/metrics | grep astradns_upstream_latency
```

**Causas comunes:**

| Condición | Causa | Solución |
|-----------|-------|----------|
| Todos los upstreams no saludables | Problema de conectividad de red | Verifique que el nodo pueda alcanzar las IPs de los upstreams |
| Alta latencia + SERVFAIL | Timeout del upstream | Aumente el timeout en la configuración del health check |
| SERVFAIL intermitente | Motor DNS sobrecargado | Aumente los límites de recursos del agent |

### El ConfigMap está vacío después de crear un pool

**Diagnóstico:**

```bash
# Check operator logs
kubectl logs -n astradns-system -l app.kubernetes.io/component=operator --tail=100

# Check pool status
kubectl get dnsupstreampool <name> -n astradns-system -o yaml
```

**Causas comunes:**

- El pool tiene una especificación inválida (verifique las condiciones de estado para `InvalidSpec`)
- El operator no está corriendo
- Problema de RBAC (el operator no puede escribir ConfigMaps)

### El DNS funciona pero las métricas de AstraDNS están en cero

**Causa:** CoreDNS no está reenviando a AstraDNS.

**Solución:** Consulte [Integración con CoreDNS](../guides/coredns-integration.md) para verificar la configuración de reenvío.

## Recopilación de Información de Depuración

Para reportes de errores, recopile esta información:

```bash
# Version info
helm list -n astradns-system
kubectl get pods -n astradns-system -o wide

# Operator logs
kubectl logs -n astradns-system -l app.kubernetes.io/component=operator --tail=200

# Agent logs
kubectl logs -n astradns-system -l app.kubernetes.io/component=agent --tail=200

# CRD status
kubectl get dnsupstreampools,dnscacheprofiles,externaldnspolicies \
  -n astradns-system -o yaml

# ConfigMap
kubectl get configmap astradns-agent-config -n astradns-system -o yaml

# CoreDNS config
kubectl get configmap coredns -n kube-system -o yaml

# Node info
kubectl get nodes -o wide
```
