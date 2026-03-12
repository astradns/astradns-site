# Referencia de Metricas

O AstraDNS expoe metricas Prometheus no endpoint `:9153/metrics` do agent.

## Metricas do Agent

### Metricas de Consulta

| Metrica | Tipo | Labels | Descricao |
|---------|------|--------|-----------|
| `astradns_queries_total` | Counter | -- | Total de consultas DNS processadas |
| `astradns_queries_by_type` | Counter | `qtype` | Consultas por tipo de registro DNS (A, AAAA, CNAME, MX, TXT, SRV, SOA, NS, PTR, CAA, HTTPS, SVCB, DNSKEY, DS, RRSIG ou OTHER) |
| `astradns_nxdomain_total` | Counter | -- | Total de respostas NXDOMAIN |
| `astradns_servfail_total` | Counter | -- | Total de respostas SERVFAIL |

### Metricas de Cache

| Metrica | Tipo | Labels | Descricao |
|---------|------|--------|-----------|
| `astradns_cache_hits_total` | Counter | -- | Acertos de cache (quando o status do cache e conhecido) |
| `astradns_cache_misses_total` | Counter | -- | Falhas de cache (quando o status do cache e conhecido) |

### Metricas de Upstream

| Metrica | Tipo | Labels | Descricao |
|---------|------|--------|-----------|
| `astradns_upstream_queries_total` | Counter | `upstream` | Consultas encaminhadas por upstream |
| `astradns_upstream_latency_seconds` | Histogram | `upstream` | Latencia de consulta por upstream |
| `astradns_upstream_failures_total` | Counter | `upstream` | Consultas com falha por upstream |
| `astradns_upstream_healthy` | Gauge | `upstream` | Status de saude do upstream (1=saudavel, 0=nao saudavel) |
| `astradns_timeout_total` | Counter | `upstream` | Contagem de timeouts por upstream |

**Buckets do histograma:** 1ms, 5ms, 10ms, 25ms, 50ms, 100ms, 250ms, 500ms, 1s

### Metricas do Ciclo de Vida do Agent

| Metrica | Tipo | Labels | Descricao |
|---------|------|--------|-----------|
| `astradns_agent_up` | Gauge | -- | Status de execucao do agent (1=ativo, 0=encerrando) |
| `astradns_agent_config_reload_total` | Counter | -- | Total de tentativas de reload de configuracao |
| `astradns_agent_config_reload_errors_total` | Counter | -- | Tentativas de reload de configuracao com falha |
| `astradns_proxy_dropped_events_total` | Counter | -- | Eventos descartados no canal do proxy (contrapressao) |
| `astradns_fanout_dropped_events_total` | Counter | -- | Eventos descartados durante fan-out para workers |

## Consultas de Exemplo

### Taxa de Consultas

```promql
rate(astradns_queries_total[5m])
```

### Taxa de Acerto de Cache

```promql
rate(astradns_cache_hits_total[5m])
/
(rate(astradns_cache_hits_total[5m]) + rate(astradns_cache_misses_total[5m]))
```

### Latencia P95 do Upstream

```promql
histogram_quantile(0.95,
  rate(astradns_upstream_latency_seconds_bucket[5m])
)
```

### Upstreams Nao Saudaveis

```promql
astradns_upstream_healthy == 0
```

### Taxa de SERVFAIL

```promql
rate(astradns_servfail_total[5m]) / rate(astradns_queries_total[5m])
```

### Falhas de Reload de Configuracao

```promql
rate(astradns_agent_config_reload_errors_total[5m]) > 0
```

## Configuracao de Coleta

### ServiceMonitor (Prometheus Operator)

Habilite nos valores do Helm:

```yaml
serviceMonitor:
  enabled: true
```

### Configuracao Manual de Scrape

```yaml
scrape_configs:
  - job_name: astradns-agent
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_component]
        regex: agent
        action: keep
      - source_labels: [__meta_kubernetes_pod_ip]
        target_label: __address__
        replacement: "${1}:9153"
```
