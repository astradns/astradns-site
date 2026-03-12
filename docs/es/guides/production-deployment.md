# Despliegue en Producción

Esta guía cubre la configuración recomendada y los pasos de validación para desplegar AstraDNS en producción.

## Lista de Verificación Pre-Despliegue

- [ ] cert-manager instalado y ClusterIssuer configurado
- [ ] Prometheus Operator instalado (para ServiceMonitor)
- [ ] Grafana con provisionamiento por sidecar (para dashboards)
- [ ] Imágenes de contenedores disponibles en su registro
- [ ] El equipo de DNS revisó la lista de resolvers upstream
- [ ] Runbook compartido con el equipo de guardia

## Valores Recomendados

```yaml
operator:
  replicas: 1
  leaderElect: true
  tolerations:
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule

agent:
  engineType: unbound
  network:
    mode: linkLocal
  logMode: sampled
  logSampleRate: "0.1"
  tolerations:
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule

webhook:
  enabled: true
  certManager:
    issuerRef:
      name: your-cluster-issuer
      kind: ClusterIssuer

serviceMonitor:
  enabled: true

grafana:
  dashboards:
    enabled: true

coredns:
  integration:
    enabled: true

crds:
  install: true
```

## Pasos de Validación

### 1. Pods en Ejecución

```bash
kubectl get pods -n astradns-system
```

Tanto los pods del operator como del agent deben estar en estado `Running`. El agent debe tener un pod por nodo.

### 2. CRDs Registrados

```bash
kubectl get crds | grep astradns
```

### 3. Crear y Verificar el Pool de Upstreams

```bash
kubectl apply -f - <<EOF
apiVersion: dns.astradns.com/v1alpha1
kind: DNSUpstreamPool
metadata:
  name: production
  namespace: astradns-system
spec:
  upstreams:
    - address: "1.1.1.1"
    - address: "8.8.8.8"
  healthCheck:
    enabled: true
EOF
```

```bash
# Should show Ready=True
kubectl get dnsupstreampools -n astradns-system -o wide
```

### 4. Verificar el ConfigMap

```bash
kubectl get configmap astradns-agent-config -n astradns-system \
  -o jsonpath='{.data.config\.json}' | jq .
```

### 5. Probar la Resolución DNS

```bash
kubectl run dns-test --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup example.com
```

### 6. Verificar Métricas

```bash
kubectl port-forward -n astradns-system ds/astradns-agent 9153:9153 &
curl -s http://localhost:9153/metrics | grep astradns_queries_total
```

### 7. Verificar Endpoints de Salud

```bash
kubectl port-forward -n astradns-system ds/astradns-agent 8080:8080 &
curl -s http://localhost:8080/healthz   # Should return 200
curl -s http://localhost:8080/readyz    # Should return 200
```

## Objetivos de SLO

| SLO | Objetivo | Medición |
|-----|----------|----------|
| Tiempo de instalación | < 5 minutos | Tiempo desde `helm install` hasta que todos los pods estén Running |
| Latencia DNS p95 | <= línea base | `histogram_quantile(0.95, rate(astradns_upstream_latency_seconds_bucket[5m]))` |
| Ratio de aciertos de caché | > 30% | `rate(astradns_cache_hits_total[1h]) / (hits + misses)` |
| Tiempo de recuperación | < 30 segundos | Tiempo desde el reinicio del pod hasta Ready |
| Fallas inducidas por AstraDNS | 0 | `rate(astradns_servfail_total[5m])` con upstreams saludables |

Ejecute la validación automatizada de SLOs:

```bash
make test-slo
```

## Monitoreo y Alertas

### Alertas Recomendadas

| Alerta | Condición | Severidad |
|--------|-----------|-----------|
| AgentDown | `astradns_agent_up == 0` durante 2m | Crítica |
| UpstreamUnhealthy | `astradns_upstream_healthy == 0` durante 5m | Advertencia |
| HighServfailRate | `rate(astradns_servfail_total[5m]) / rate(astradns_queries_total[5m]) > 0.05` | Advertencia |
| ConfigReloadFailing | `rate(astradns_agent_config_reload_errors_total[5m]) > 0` | Advertencia |
| EventsDropped | `rate(astradns_proxy_dropped_events_total[5m]) > 0` | Informativa |

### Dashboard

El dashboard de Grafana (habilitado mediante `grafana.dashboards.enabled=true`) muestra:

- Tasa de consultas y latencia (p50, p95, p99)
- Ratio de aciertos de caché a lo largo del tiempo
- Estado de salud de los upstreams
- Tasas de SERVFAIL y NXDOMAIN
- Eventos del ciclo de vida del agent (recargas, reinicios)
