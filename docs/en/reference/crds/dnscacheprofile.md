# DNSCacheProfile

Configures DNS cache behavior including size limits, TTL bounds, and prefetch thresholds.

## Example

```yaml
apiVersion: dns.astradns.com/v1alpha1
kind: DNSCacheProfile
metadata:
  name: default
  namespace: astradns-system
spec:
  maxEntries: 50000
  positiveTtl:
    minSeconds: 60
    maxSeconds: 300
  negativeTtl:
    seconds: 30
  prefetch:
    enabled: true
    threshold: 5
```

!!! note "Automatic discovery"
    The operator automatically uses the cache profile named `default` in the same namespace as the upstream pool. No explicit reference is needed.

## Spec

### `maxEntries` (optional)

| Field | Type | Default | Validation | Description |
|-------|------|---------|------------|-------------|
| `maxEntries` | int32 | `100000` | min 1 | Maximum number of entries in the DNS cache |

### `positiveTtl` (optional)

Controls TTL clamping for successful (NOERROR) responses.

| Field | Type | Default | Validation | Description |
|-------|------|---------|------------|-------------|
| `minSeconds` | int32 | `60` | min 1 | Minimum TTL applied to cached responses |
| `maxSeconds` | int32 | `300` | min 1, must be >= minSeconds | Maximum TTL applied to cached responses |

!!! info "Cross-field validation"
    The CRD enforces `maxSeconds >= minSeconds` via CEL validation. Creating a profile with `maxSeconds < minSeconds` is rejected at admission time.

### `negativeTtl` (optional)

| Field | Type | Default | Validation | Description |
|-------|------|---------|------------|-------------|
| `seconds` | int32 | `30` | min 1 | TTL for negative (NXDOMAIN) responses |

### `prefetch` (optional)

Prefetching re-resolves popular domains before their TTL expires, keeping them in cache.

| Field | Type | Default | Validation | Description |
|-------|------|---------|------------|-------------|
| `enabled` | bool | `true` | — | Enable prefetch for popular domains |
| `threshold` | int32 | `10` | min 1 | Number of queries before a domain is considered popular |

## Status

| Field | Type | Description |
|-------|------|-------------|
| `conditions` | []Condition | Standard conditions |

### Conditions

| Type | Status | Reason | Meaning |
|------|--------|--------|---------|
| `Active` | `True` | `Valid` | Profile validated successfully |
