# Referencia de Métricas

AstraDNS expone métricas de Prometheus en el endpoint `:9153/metrics` del agent.

## Métricas del Agent

### Métricas de Consultas

| Métrica | Tipo | Etiquetas | Descripción |
|---------|------|-----------|-------------|
| `astradns_queries_total` | Counter | -- | Total de consultas DNS procesadas |
| `astradns_queries_by_type` | Counter | `qtype` | Consultas por tipo de registro DNS (A, AAAA, CNAME, MX, TXT, SRV, SOA, NS, PTR, CAA, HTTPS, SVCB, DNSKEY, DS, RRSIG u OTHER) |
| `astradns_nxdomain_total` | Counter | -- | Total de respuestas NXDOMAIN |
| `astradns_servfail_total` | Counter | -- | Total de respuestas SERVFAIL |

### Métricas de Caché

| Métrica | Tipo | Etiquetas | Descripción |
|---------|------|-----------|-------------|
| `astradns_cache_hits_total` | Counter | -- | Aciertos de caché (cuando el estado del caché es conocido) |
| `astradns_cache_misses_total` | Counter | -- | Fallos de caché (cuando el estado del caché es conocido) |

### Métricas de Upstream

| Métrica | Tipo | Etiquetas | Descripción |
|---------|------|-----------|-------------|
| `astradns_upstream_queries_total` | Counter | `upstream` | Consultas reenviadas por upstream |
| `astradns_upstream_latency_seconds` | Histogram | `upstream` | Latencia de consulta por upstream |
| `astradns_upstream_failures_total` | Counter | `upstream` | Consultas fallidas por upstream |
| `astradns_upstream_healthy` | Gauge | `upstream` | Estado de salud del upstream (1=saludable, 0=no saludable) |
| `astradns_timeout_total` | Counter | `upstream` | Conteo de timeouts por upstream |

**Buckets del histograma:** 1ms, 5ms, 10ms, 25ms, 50ms, 100ms, 250ms, 500ms, 1s

### Métricas del Ciclo de Vida del Agent

| Métrica | Tipo | Etiquetas | Descripción |
|---------|------|-----------|-------------|
| `astradns_agent_up` | Gauge | -- | Estado de ejecución del agent (1=activo, 0=apagándose) |
| `astradns_agent_config_reload_total` | Counter | -- | Total de intentos de recarga de configuración |
| `astradns_agent_config_reload_errors_total` | Counter | -- | Intentos fallidos de recarga de configuración |
| `astradns_proxy_dropped_events_total` | Counter | -- | Eventos descartados en el canal del proxy (contrapresión) |
| `astradns_fanout_dropped_events_total` | Counter | -- | Eventos descartados durante el fan-out a workers |

## Consultas de Ejemplo

### Tasa de Consultas

```promql
rate(astradns_queries_total[5m])
```

### Ratio de Aciertos de Caché

```promql
rate(astradns_cache_hits_total[5m])
/
(rate(astradns_cache_hits_total[5m]) + rate(astradns_cache_misses_total[5m]))
```

### Latencia P95 del Upstream

```promql
histogram_quantile(0.95,
  rate(astradns_upstream_latency_seconds_bucket[5m])
)
```

### Upstreams No Saludables

```promql
astradns_upstream_healthy == 0
```

### Tasa de SERVFAIL

```promql
rate(astradns_servfail_total[5m]) / rate(astradns_queries_total[5m])
```

### Fallas de Recarga de Configuración

```promql
rate(astradns_agent_config_reload_errors_total[5m]) > 0
```

## Configuración de Scraping

### ServiceMonitor (Prometheus Operator)

Habilitar en los valores de Helm:

```yaml
serviceMonitor:
  enabled: true
```

### Configuración de Scrape Manual

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
