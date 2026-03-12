# CoreDNS Integration

This guide explains how AstraDNS patches CoreDNS to forward external DNS traffic through the Agent, with profile-aware behavior for `node-local` and `central`.

---

## How it works

When `clusterDNS.forwardExternalToAstraDNS.enabled=true`, the chart creates a post-install/post-upgrade Job that:

1. reads the current CoreDNS `Corefile` from the ConfigMap;
2. stores a backup into `Corefile.astradns.backup`;
3. rewrites the `forward` directive to point to AstraDNS;
4. optionally restarts the CoreDNS deployment.

---

## Profile-specific configuration

### node-local

```yaml
agent:
  topology:
    profile: node-local
  network:
    mode: linkLocal
    linkLocalIP: 169.254.20.11

clusterDNS:
  forwardExternalToAstraDNS:
    enabled: true
    namespace: kube-system
    configMapName: coredns
    rolloutDeployment: coredns
    forwardTarget: 169.254.20.11:5353
    fallbackUpstream: /etc/resolv.conf
```

### central with fixed ClusterIP

```yaml
agent:
  topology:
    profile: central
  deployment:
    replicas: 3
  dnsService:
    type: ClusterIP
    clusterIP: 10.96.0.53
    port: 53

clusterDNS:
  forwardExternalToAstraDNS:
    enabled: true
    namespace: kube-system
    configMapName: coredns
    rolloutDeployment: coredns
    fallbackUpstream: /etc/resolv.conf
```

### central with service IP auto-discovery

```yaml
agent:
  topology:
    profile: central
  dnsService:
    type: ClusterIP
    clusterIP: ""
    port: 53

clusterDNS:
  forwardExternalToAstraDNS:
    enabled: true
    namespace: kube-system
    configMapName: coredns
```

In this mode, the patch job queries the Agent DNS Service at runtime and injects the discovered `clusterIP` into CoreDNS.

---

## Resulting `forward` block

With fallback enabled, CoreDNS uses a pattern like:

```text
forward . <astradns-target> /etc/resolv.conf {
    policy sequential
}
```

`policy sequential` keeps AstraDNS as primary and automatically falls back to the node resolver if AstraDNS becomes unreachable.

---

## Guardrails

- `node-local` + CoreDNS patching requires `agent.network.mode=linkLocal`.
- `central` rejects `agent.network.mode=linkLocal`.
- `clusterDNS.provider` currently supports only `coredns`.
- `kubectlImagePullPolicy` must be `Always`, `IfNotPresent`, or `Never`.

---

## Post-deploy validation

```bash
# 1) Final Corefile
kubectl -n kube-system get configmap coredns -o jsonpath='{.data.Corefile}'

# 2) Backup presence
kubectl -n kube-system get configmap coredns -o jsonpath='{.data.Corefile\.astradns\.backup}'

# 3) Patch job execution
kubectl -n astradns-system get jobs | grep coredns-patch
kubectl -n astradns-system logs job/<release>-astradns-coredns-patch

# 4) DNS smoke test
kubectl run dns-test --rm -it --restart=Never --image=busybox:1.37 -- nslookup example.com
```

In `central`, also verify the Service endpoint:

```bash
kubectl -n astradns-system get svc <release>-astradns-agent-dns -o wide
```

---

## Manual rollback

```bash
kubectl -n kube-system patch configmap coredns --type merge --patch-file <(cat <<'EOF'
data:
  Corefile: |
    # paste content from Corefile.astradns.backup
EOF
)

kubectl -n kube-system rollout restart deployment coredns
```

---

## Related

- [Topology Profiles](topology-profiles.md)
- [Runbook](../operations/runbook.md)
