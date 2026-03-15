# Implantacao em Producao

Este guia cobre a configuracao recomendada e os passos de validacao para implantar o AstraDNS em producao.

## Checklist Pre-Implantacao

- [ ] cert-manager instalado e ClusterIssuer configurado
- [ ] Prometheus Operator instalado (para ServiceMonitor)
- [ ] Grafana com provisionamento via sidecar (para dashboards)
- [ ] Imagens de container disponiveis no seu registry
- [ ] Equipe de DNS revisou a lista de resolvers upstream
- [ ] Runbook compartilhado com a equipe de plantao

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
  topology:
    profile: node-local
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

clusterDNS:
  forwardExternalToAstraDNS:
    enabled: true

crds:
  install: true
```

## Passos de Validacao

### 1. Pods em Execucao

```bash
kubectl get pods -n astradns-system
```

Tanto os pods do operator quanto do agent devem estar `Running`. O agent deve ter um pod por no.

### 2. CRDs Registrados

```bash
kubectl get crds | grep astradns
```

### 3. Criar e Verificar o Pool de Upstreams

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
# Deve exibir Ready=True
kubectl get dnsupstreampools -n astradns-system -o wide
```

### 4. Verificar o ConfigMap

```bash
kubectl get configmap astradns-agent-config -n astradns-system \
  -o jsonpath='{.data.config\.json}' | jq .
```

### 5. Testar a Resolucao DNS

```bash
kubectl run dns-test --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup example.com
```

### 6. Verificar as Metricas

```bash
AGENT_POD="$(kubectl -n astradns-system get pods -l app.kubernetes.io/component=agent -o jsonpath='{.items[0].metadata.name}')"
kubectl port-forward -n astradns-system "pod/${AGENT_POD}" 9153:9153 &
curl -s http://localhost:9153/metrics | grep astradns_queries_total
```

### 7. Verificar os Endpoints de Saude

```bash
kubectl port-forward -n astradns-system "pod/${AGENT_POD}" 8080:8080 &
curl -s http://localhost:8080/healthz   # Deve retornar 200
curl -s http://localhost:8080/readyz    # Deve retornar 200
```

## Metas de SLO

| SLO | Meta | Medicao |
|-----|------|---------|
| Tempo de instalacao | < 5 minutos | Tempo desde `helm upgrade --install` ate todos os pods Running |
| Latencia DNS p95 | <= baseline | `histogram_quantile(0.95, rate(astradns_upstream_latency_seconds_bucket[5m]))` |
| Taxa de acerto de cache | > 30% | `rate(astradns_cache_hits_total[1h]) / (hits + misses)` |
| Tempo de recuperacao | < 30 segundos | Tempo desde o reinicio do pod ate Ready |
| Falhas induzidas pelo AstraDNS | 0 | `rate(astradns_servfail_total[5m])` com upstreams saudaveis |

Execute a validacao automatizada de SLO:

```bash
make test-slo
```

## Monitoramento e Alertas

### Alertas Recomendados

| Alerta | Condicao | Severidade |
|--------|----------|------------|
| AgentDown | `astradns_agent_up == 0` por 2m | Critico |
| UpstreamUnhealthy | `astradns_upstream_healthy == 0` por 5m | Aviso |
| HighServfailRate | `rate(astradns_servfail_total[5m]) / rate(astradns_queries_total[5m]) > 0.05` | Aviso |
| ConfigReloadFailing | `rate(astradns_agent_config_reload_errors_total[5m]) > 0` | Aviso |
| EventsDropped | `rate(astradns_proxy_dropped_events_total[5m]) > 0` | Informativo |

### Dashboard

O dashboard Grafana (habilitado via `grafana.dashboards.enabled=true`) exibe:

- Taxa de consultas e latencia (p50, p95, p99)
- Taxa de acerto de cache ao longo do tempo
- Status de saude dos upstreams
- Taxas de SERVFAIL e NXDOMAIN
- Eventos do ciclo de vida do agent (reloads, reinicializacoes)
