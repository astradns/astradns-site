# DNSUpstreamPool

Defines a pool of upstream DNS resolvers with health checking and load balancing configuration.

## Example

```yaml
apiVersion: dns.astradns.com/v1alpha1
kind: DNSUpstreamPool
metadata:
  name: production
  namespace: astradns-system
spec:
  upstreams:
    - address: "1.1.1.1"
      port: 53
    - address: "8.8.8.8"
      port: 53
    - address: "9.9.9.9"
  healthCheck:
    enabled: true
    intervalSeconds: 30
    timeoutSeconds: 5
    failureThreshold: 3
  loadBalancing:
    strategy: round-robin
```

## Spec

### `upstreams` (required)

List of upstream DNS resolvers. At least one upstream is required.

| Field | Type | Default | Validation | Description |
|-------|------|---------|------------|-------------|
| `address` | string | — | Required, min length 1 | IP address or hostname of the upstream resolver |
| `port` | int32 | `53` | 1-65535 | Port number |

### `healthCheck` (optional)

| Field | Type | Default | Validation | Description |
|-------|------|---------|------------|-------------|
| `enabled` | bool | `true` | — | Enable periodic health probing |
| `intervalSeconds` | int32 | `30` | min 1 | Seconds between health checks |
| `timeoutSeconds` | int32 | `5` | min 1 | Timeout for each probe |
| `failureThreshold` | int32 | `3` | min 1 | Consecutive failures before marking unhealthy |

### `loadBalancing` (optional)

| Field | Type | Default | Validation | Description |
|-------|------|---------|------------|-------------|
| `strategy` | string | `round-robin` | Enum: `round-robin`, `first-available`, `random` | Load balancing strategy across upstreams |

## Status

| Field | Type | Description |
|-------|------|-------------|
| `conditions` | []Condition | Standard conditions (see below) |
| `upstreamStatuses` | []UpstreamStatus | Per-upstream health and latency |

### Conditions

| Type | Status | Reason | Meaning |
|------|--------|--------|---------|
| `Ready` | `True` | `Ready` | Configuration rendered successfully |
| `Ready` | `False` | `Superseded` | Another pool is active in this namespace |
| `Ready` | `False` | `InvalidSpec` | Spec validation failed |
| `Ready` | `False` | `ConfigGenerationFailed` | Engine config generation error |
| `Ready` | `False` | `ValidationFailed` | Rendered config failed validation |
| `Ready` | `False` | `ProfileLookupFailed` | Could not fetch default cache profile |

### UpstreamStatus

| Field | Type | Description |
|-------|------|-------------|
| `address` | string | Upstream address |
| `healthy` | bool | Current health status |
| `latencyMs` | int64 | Last probe latency in milliseconds |

## Multi-Pool Behavior

Only **one pool per namespace** is active at a time. When multiple pools exist, the active pool is selected by:

1. **Oldest `creationTimestamp`** — the pool created first wins
2. **Lowest initial ResourceVersion** — tiebreaker for pools created in the same second
3. **Alphabetical name** — ultimate deterministic tiebreaker

Non-active pools receive the `Superseded` condition with a message indicating which pool is active.

!!! tip "Validating webhook"
    Enable the validating webhook (`webhook.enabled=true`) to reject creation of a second pool in the same namespace at admission time.
