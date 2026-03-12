# CoreDNS Integration

AstraDNS integrates with the cluster's CoreDNS to intercept external DNS queries. This guide explains how the integration works and how to configure it.

## How It Works

When `coredns.integration.enabled=true`, the Helm chart creates a post-install Job that:

1. Creates RBAC (ServiceAccount, Role, RoleBinding) in `kube-system`
2. Reads the current CoreDNS ConfigMap
3. Backs up the original Corefile to a new ConfigMap
4. Adds a `forward` directive that routes non-cluster queries to AstraDNS
5. Restarts the CoreDNS deployment to pick up the change

### Before Integration

```
.:53 {
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
    }
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

### After Integration

```
.:53 {
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
    }
    forward . 169.254.20.11:5353 /etc/resolv.conf {
        policy sequential
    }
    cache 30
    loop
    reload
    loadbalance
}
```

With `policy sequential`, CoreDNS always tries the AstraDNS agent first. If the agent becomes unreachable, CoreDNS automatically fails over to `/etc/resolv.conf` (the node's original resolver) within ~1 second. When the agent recovers, traffic flows back automatically.

## Configuration

### Enable via Helm

```yaml
agent:
  network:
    mode: linkLocal                   # Required for CoreDNS integration
    linkLocalIP: 169.254.20.11

clusterDNS:
  forwardExternalToAstraDNS:
    enabled: true
    fallbackUpstream: /etc/resolv.conf  # Default — automatic failover
```

To disable the fallback (not recommended):

```yaml
clusterDNS:
  forwardExternalToAstraDNS:
    enabled: true
    fallbackUpstream: ""
```

### Required Network Mode

CoreDNS integration requires `linkLocal` network mode. The agent binds to `169.254.20.11:53` on each node, providing a consistent address for CoreDNS forwarding.

!!! warning
    Using `hostPort` mode with CoreDNS integration is not supported because CoreDNS would need to know each node's IP address, which varies across the cluster.

## Failover Behavior

By default, the patch job configures CoreDNS with a fallback upstream (`/etc/resolv.conf`) using `policy sequential`:

| Scenario | Behavior |
|----------|----------|
| Agent healthy | All external queries go through AstraDNS (metrics, filtering, caching) |
| Agent unreachable | CoreDNS detects failure within ~1s and routes to fallback upstream |
| Agent recovers | CoreDNS health check restores AstraDNS as primary within ~10s |

!!! note
    During failover, queries bypass AstraDNS — no metrics, no domain filtering, no proxy cache. The DaemonSet ensures the agent pod restarts automatically, minimizing failover duration.

## Verification

After installation, verify the integration:

```bash
# Check the CoreDNS ConfigMap was patched
kubectl get configmap coredns -n kube-system -o yaml | grep forward

# Check the backup exists
kubectl get configmap coredns-backup-astradns -n kube-system

# Test DNS resolution through AstraDNS
kubectl run dns-test --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup example.com
```

Check agent metrics to confirm queries are flowing:

```bash
kubectl port-forward -n astradns-system ds/astradns-agent 9153:9153
curl -s http://localhost:9153/metrics | grep astradns_queries_total
```

## Rollback

To manually revert the CoreDNS integration:

```bash
# Restore from backup
kubectl get configmap coredns-backup-astradns -n kube-system \
  -o jsonpath='{.data.Corefile}' | \
  kubectl create configmap coredns -n kube-system \
  --from-file=Corefile=/dev/stdin --dry-run=client -o yaml | \
  kubectl apply -f -

# Restart CoreDNS
kubectl rollout restart deployment coredns -n kube-system
```

Or simply uninstall AstraDNS — the `helm uninstall` process does not automatically revert CoreDNS changes, so manual rollback may be needed.
