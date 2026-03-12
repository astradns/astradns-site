# CRD Reference

AstraDNS defines three Custom Resource Definitions in the `dns.astradns.com/v1alpha1` API group.

| CRD | Short Description | Scope |
|-----|-------------------|-------|
| [`DNSUpstreamPool`](dnsupstreampool.md) | Upstream resolvers, health checks, load balancing | Namespaced |
| [`DNSCacheProfile`](dnscacheprofile.md) | Cache size, TTL bounds, prefetch | Namespaced |
| [`ExternalDNSPolicy`](externaldnspolicy.md) | Namespace-to-pool mapping | Namespaced |

## API Group

```
dns.astradns.com/v1alpha1
```

!!! warning "Alpha API"
    The `v1alpha1` API is under active development. Field names and semantics may change in future versions. See the [upgrade guide](../../operations/upgrade-guide.md) for migration instructions.

## Status Conditions

All three CRDs use the standard Kubernetes Conditions pattern:

```yaml
status:
  conditions:
    - type: Ready          # or Active, Validated
      status: "True"       # or "False"
      reason: Ready        # machine-readable reason
      message: "..."       # human-readable message
      lastTransitionTime: "2026-03-11T10:00:00Z"
      observedGeneration: 1
```
