# Upgrade Guide

## Upgrade Process

### 1. Review Release Notes

Before upgrading, review the release notes for breaking changes, new features, and deprecations.

### 2. Backup

```bash
# Backup CRDs
kubectl get dnsupstreampools,dnscacheprofiles,externaldnspolicies \
  -n astradns-system -o yaml > astradns-backup.yaml

# Backup Helm values
helm get values astradns -n astradns-system > current-values.yaml

# Note current revision
helm history astradns -n astradns-system
```

### 3. Dry Run

```bash
helm upgrade astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system \
  -f current-values.yaml \
  --dry-run --debug
```

Review the output for unexpected changes.

### 4. Upgrade

```bash
helm upgrade astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system \
  -f current-values.yaml
```

### 5. Verify

```bash
# Check pods are running
kubectl get pods -n astradns-system

# Verify CRD status
kubectl get dnsupstreampools -n astradns-system

# Test DNS resolution
kubectl run dns-test --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup example.com

# Check metrics
AGENT_POD="$(kubectl -n astradns-system get pods -l app.kubernetes.io/component=agent -o jsonpath='{.items[0].metadata.name}')"
kubectl port-forward -n astradns-system "pod/${AGENT_POD}" 9153:9153 &
curl -s http://localhost:9153/metrics | grep astradns_queries_total
```

## Rollback

If the upgrade causes issues:

```bash
helm rollback astradns <previous-revision> -n astradns-system
```

## CRD Upgrades

!!! warning "CRD changes require manual attention"
    If the new version includes CRD changes, run upgrade with CRD rendering enabled:

    ```bash
    helm upgrade astradns oci://ghcr.io/astradns/helm-charts/astradns \
      --namespace astradns-system \
      -f current-values.yaml \
      --set crds.install=true
    ```

## Version-Specific Notes

### v0.1.x

Initial release. No breaking changes within the v0.1.x series.

**Known limitations:**

- Namespace-level metrics show as "unknown" (requires direct data path in future release)
- ExternalDNSPolicy validates references but does not enforce routing
- Only one DNSUpstreamPool per namespace (by design; webhook enforces this when enabled)
