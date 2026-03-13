# SLO Validation

AstraDNS defines five Service Level Objectives that are validated before every release.

## SLO Targets

| # | SLO | Target | How to Measure |
|---|-----|--------|----------------|
| 1 | Install time | < 5 minutes | Time from `helm install` to all pods Running |
| 2 | DNS latency p95 | <= baseline without AstraDNS | `histogram_quantile(0.95, rate(astradns_upstream_latency_seconds_bucket[5m]))` |
| 3 | Cache hit ratio | > 30% after warm-up | `rate(cache_hits) / (rate(cache_hits) + rate(cache_misses))` |
| 4 | Recovery time | < 30 seconds | Time from agent pod restart to Ready |
| 5 | AstraDNS-induced failures | 0 | No SERVFAIL responses caused by AstraDNS itself |

## Running SLO Validation

### Automated

```bash
make test-slo
```

This runs the `test/slo/validate-mvp.sh` script which:

1. Verifies pod readiness
2. Measures DNS query latency baseline
3. Runs repeated queries to build cache
4. Measures cache hit ratio
5. Simulates pod restart and measures recovery time
6. Reports pass/fail for each SLO

### Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `SLO_NAMESPACE` | `astradns-system` | Namespace to test |
| `SLO_RELEASE` | `astradns` | Helm release name |
| `SLO_ITERATIONS` | `100` | Number of DNS queries per test |
| `SLO_DOMAIN` | `example.com` | Domain to query |

### Manual Validation

#### SLO 1: Install Time

```bash
time helm upgrade --install astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system --create-namespace \
  --set agent.engineType=unbound

kubectl wait --for=condition=Ready pods \
  -l app.kubernetes.io/part-of=astradns \
  -n astradns-system --timeout=300s
```

#### SLO 2: Latency

```bash
# Baseline (without AstraDNS)
kubectl run dns-baseline --rm -it --restart=Never \
  --image=busybox:1.37 -- sh -c \
  'for i in $(seq 1 100); do nslookup example.com > /dev/null 2>&1; done'

# With AstraDNS (compare times)
```

#### SLO 3: Cache Hit Ratio

```bash
# After warm-up queries
curl -s http://localhost:9153/metrics | \
  grep -E 'astradns_cache_(hits|misses)_total'
```

#### SLO 4: Recovery Time

```bash
# Delete an agent pod and measure recovery
kubectl delete pod -n astradns-system -l app.kubernetes.io/component=agent --wait=false
time kubectl wait --for=condition=Ready pods \
  -l app.kubernetes.io/component=agent \
  -n astradns-system --timeout=60s
```
