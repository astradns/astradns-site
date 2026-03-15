# Troubleshooting

## Quick Diagnostics

```bash
# One-liner health check
kubectl get pods,dnsupstreampools,dnscacheprofiles -n astradns-system
```

## Common Issues

### "No CRDs to install; skipping" during helm install

**Cause:** CRDs are not being installed.

**Fix:** Ensure `crds.install=true` in your values:

```bash
helm upgrade --install astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system --create-namespace \
  --set crds.install=true
```

### Agent pod is CrashLoopBackOff

**Diagnosis:**

```bash
kubectl logs -n astradns-system -l app.kubernetes.io/component=agent --previous
kubectl describe pod -n astradns-system -l app.kubernetes.io/component=agent
```

**Common causes:**

| Log message | Cause | Fix |
|-------------|-------|-----|
| `exec: "unbound": not found` | Wrong engine type for image | Match `agent.engineType` to the engine binary in the image |
| `listen tcp :5353: bind: address already in use` | Port conflict | Check for other services on port 5353 |
| `open /etc/astradns/config/config.json: no such file` | ConfigMap not mounted | Verify ConfigMap exists and is referenced in the agent workload (DaemonSet/Deployment) |

### Pool shows Ready=False, Reason=Superseded

**Cause:** Multiple `DNSUpstreamPool` resources exist in the namespace. Only the oldest pool is active.

**Fix:**

```bash
# See all pools and their status
kubectl get dnsupstreampools -n astradns-system -o yaml

# Delete the superseded pool
kubectl delete dnsupstreampool <superseded-name> -n astradns-system
```

### Metrics show zero cache hits

**Possible causes:**

1. **Cache not warm:** Run repeated queries to the same domain and check again
2. **Cache disabled:** Verify `DNSCacheProfile` exists and `maxEntries > 0`
3. **Proxy limitation:** The proxy reports cache status as "unknown" — cache metrics come from the engine, not the proxy

### High SERVFAIL rate

**Diagnosis:**

```bash
# Check upstream health
curl -s http://localhost:9153/metrics | grep astradns_upstream_healthy

# Check upstream latency
curl -s http://localhost:9153/metrics | grep astradns_upstream_latency
```

**Common causes:**

| Condition | Cause | Fix |
|-----------|-------|-----|
| All upstreams unhealthy | Network connectivity issue | Verify node can reach upstream IPs |
| High latency + SERVFAIL | Upstream timeout | Increase timeout in health check config |
| Intermittent SERVFAIL | DNS engine overloaded | Increase agent resource limits |

### ConfigMap is empty after creating a pool

**Diagnosis:**

```bash
# Check operator logs
kubectl logs -n astradns-system -l app.kubernetes.io/component=operator --tail=100

# Check pool status
kubectl get dnsupstreampool <name> -n astradns-system -o yaml
```

**Common causes:**

- Pool has invalid spec (check status conditions for `InvalidSpec`)
- Operator is not running
- RBAC issue (operator can't write ConfigMaps)

### DNS works but AstraDNS metrics are zero

**Cause:** CoreDNS is not forwarding to AstraDNS.

**Fix:** See [CoreDNS Integration](../guides/coredns-integration.md) to verify the forwarding configuration.

## Collecting Debug Information

For bug reports, collect this information:

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
