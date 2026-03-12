# Production Deployment

This guide covers the recommended configuration and validation steps for deploying AstraDNS in production.

## Pre-Deployment Checklist

- [ ] cert-manager installed and ClusterIssuer configured
- [ ] Prometheus Operator installed (for ServiceMonitor)
- [ ] Grafana with sidecar provisioning (for dashboards)
- [ ] Container images available in your registry
- [ ] DNS team reviewed upstream resolver list
- [ ] Runbook shared with on-call team

## Recommended Values

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

## Validation Steps

### 1. Pods Running

```bash
kubectl get pods -n astradns-system
```

Both operator and agent pods should be `Running`. Agent should have one pod per node.

### 2. CRDs Registered

```bash
kubectl get crds | grep astradns
```

### 3. Create and Verify Upstream Pool

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

### 4. Verify ConfigMap

```bash
kubectl get configmap astradns-agent-config -n astradns-system \
  -o jsonpath='{.data.config\.json}' | jq .
```

### 5. Test DNS Resolution

```bash
kubectl run dns-test --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup example.com
```

### 6. Verify Metrics

```bash
kubectl port-forward -n astradns-system ds/astradns-agent 9153:9153 &
curl -s http://localhost:9153/metrics | grep astradns_queries_total
```

### 7. Verify Health Endpoints

```bash
kubectl port-forward -n astradns-system ds/astradns-agent 8080:8080 &
curl -s http://localhost:8080/healthz   # Should return 200
curl -s http://localhost:8080/readyz    # Should return 200
```

## SLO Targets

| SLO | Target | Measurement |
|-----|--------|-------------|
| Install time | < 5 minutes | Time from `helm install` to all pods Running |
| DNS latency p95 | <= baseline | `histogram_quantile(0.95, rate(astradns_upstream_latency_seconds_bucket[5m]))` |
| Cache hit ratio | > 30% | `rate(astradns_cache_hits_total[1h]) / (hits + misses)` |
| Recovery time | < 30 seconds | Time from pod restart to Ready |
| AstraDNS-induced failures | 0 | `rate(astradns_servfail_total[5m])` with healthy upstreams |

Run the automated SLO validation:

```bash
make test-slo
```

## Monitoring and Alerting

### Recommended Alerts

| Alert | Condition | Severity |
|-------|-----------|----------|
| AgentDown | `astradns_agent_up == 0` for 2m | Critical |
| UpstreamUnhealthy | `astradns_upstream_healthy == 0` for 5m | Warning |
| HighServfailRate | `rate(astradns_servfail_total[5m]) / rate(astradns_queries_total[5m]) > 0.05` | Warning |
| ConfigReloadFailing | `rate(astradns_agent_config_reload_errors_total[5m]) > 0` | Warning |
| EventsDropped | `rate(astradns_proxy_dropped_events_total[5m]) > 0` | Info |

### Dashboard

The Grafana dashboard (enabled via `grafana.dashboards.enabled=true`) shows:

- Query rate and latency (p50, p95, p99)
- Cache hit ratio over time
- Upstream health status
- SERVFAIL and NXDOMAIN rates
- Agent lifecycle events (reloads, restarts)
