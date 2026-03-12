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
helm upgrade astradns deploy/helm/astradns \
  --namespace astradns-system \
  -f current-values.yaml \
  --dry-run --debug
```

Review the output for unexpected changes.

### 4. Upgrade

```bash
helm upgrade astradns deploy/helm/astradns \
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
kubectl port-forward -n astradns-system ds/astradns-agent 9153:9153 &
curl -s http://localhost:9153/metrics | grep astradns_queries_total
```

## Rollback

If the upgrade causes issues:

```bash
helm rollback astradns <previous-revision> -n astradns-system
```

## CRD Upgrades

!!! warning "CRD changes require manual attention"
    Helm does not update CRDs on `helm upgrade`. If the new version includes CRD changes, apply them manually:

    ```bash
    kubectl apply -f deploy/helm/astradns/crds/
    ```

## Version-Specific Notes

### v0.1.x

Initial release. No breaking changes within the v0.1.x series.

**Known limitations:**

- Namespace-level metrics show as "unknown" (requires direct data path in future release)
- ExternalDNSPolicy validates references but does not enforce routing
- Only one DNSUpstreamPool per namespace (by design; webhook enforces this when enabled)
