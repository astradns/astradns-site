# Metrics Reference

AstraDNS exposes Prometheus metrics on the agent's `:9153/metrics` endpoint.

## Agent Metrics

### Query Metrics

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `astradns_queries_total` | Counter | — | Total DNS queries processed |
| `astradns_queries_by_type` | Counter | `qtype` | Queries by DNS record type (A, AAAA, CNAME, MX, TXT, SRV, SOA, NS, PTR, CAA, HTTPS, SVCB, DNSKEY, DS, RRSIG, or OTHER) |
| `astradns_nxdomain_total` | Counter | — | Total NXDOMAIN responses |
| `astradns_servfail_total` | Counter | — | Total SERVFAIL responses |
| `astradns_denied_queries_total` | Counter | — | Queries denied by domain filter rules |

### Cache Metrics

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `astradns_cache_hits_total` | Counter | — | Cache hits (when cache status is known) |
| `astradns_cache_misses_total` | Counter | — | Cache misses (when cache status is known) |

### Upstream Metrics

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `astradns_upstream_queries_total` | Counter | `upstream` | Queries forwarded per upstream |
| `astradns_upstream_latency_seconds` | Histogram | `upstream` | Query latency per upstream |
| `astradns_upstream_failures_total` | Counter | `upstream` | Failed queries per upstream |
| `astradns_upstream_healthy` | Gauge | `upstream` | Upstream health status (1=healthy, 0=unhealthy) |
| `astradns_timeout_total` | Counter | `upstream` | Timeout count per upstream |

**Histogram buckets:** 1ms, 5ms, 10ms, 25ms, 50ms, 100ms, 250ms, 500ms, 1s

### Agent Lifecycle Metrics

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `astradns_agent_up` | Gauge | — | Agent running status (1=up, 0=shutting down) |
| `astradns_agent_config_reload_total` | Counter | — | Total config reload attempts |
| `astradns_agent_config_reload_errors_total` | Counter | — | Failed config reload attempts |
| `astradns_proxy_dropped_events_total` | Counter | — | Events dropped at proxy channel (backpressure) |
| `astradns_fanout_dropped_events_total` | Counter | — | Events dropped during fan-out to workers |

## Example Queries

### Query Rate

```promql
rate(astradns_queries_total[5m])
```

### Cache Hit Ratio

```promql
rate(astradns_cache_hits_total[5m])
/
(rate(astradns_cache_hits_total[5m]) + rate(astradns_cache_misses_total[5m]))
```

### P95 Upstream Latency

```promql
histogram_quantile(0.95,
  rate(astradns_upstream_latency_seconds_bucket[5m])
)
```

### Unhealthy Upstreams

```promql
astradns_upstream_healthy == 0
```

### SERVFAIL Rate

```promql
rate(astradns_servfail_total[5m]) / rate(astradns_queries_total[5m])
```

### Denied Query Rate

```promql
rate(astradns_denied_queries_total[5m])
```

### Config Reload Failures

```promql
rate(astradns_agent_config_reload_errors_total[5m]) > 0
```

## Scraping Configuration

### ServiceMonitor (Prometheus Operator)

Enable in Helm values:

```yaml
serviceMonitor:
  enabled: true
```

### Manual Scrape Config

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
