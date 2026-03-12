# ExternalDNSPolicy

Maps namespaces to upstream pools and cache profiles, enabling per-namespace DNS resolution policies.

## Example

```yaml
apiVersion: dns.astradns.com/v1alpha1
kind: ExternalDNSPolicy
metadata:
  name: production-policy
  namespace: astradns-system
spec:
  selector:
    namespaces:
      - production
      - staging
  upstreamPoolRef:
    name: production
  cacheProfileRef:
    name: default
```

!!! warning "Enforcement scope"
    In the current version, `ExternalDNSPolicy` validates references and sets status conditions but does not enforce namespace-level routing. Full enforcement requires the direct data path (planned for a future release).

## Spec

### `selector` (required)

| Field | Type | Default | Validation | Description |
|-------|------|---------|------------|-------------|
| `namespaces` | []string | — | Required, min 1 item | List of namespace names this policy applies to |

### `upstreamPoolRef` (required)

| Field | Type | Default | Validation | Description |
|-------|------|---------|------------|-------------|
| `name` | string | — | Required, min length 1 | Name of the `DNSUpstreamPool` to use |

### `cacheProfileRef` (optional)

| Field | Type | Default | Validation | Description |
|-------|------|---------|------------|-------------|
| `name` | string | — | min length 1 | Name of the `DNSCacheProfile` to use |

## Status

| Field | Type | Description |
|-------|------|-------------|
| `conditions` | []Condition | Standard conditions |
| `appliedNodes` | int32 | Number of nodes where this policy is active |

### Conditions

| Type | Status | Reason | Meaning |
|------|--------|--------|---------|
| `Validated` | `True` | `Valid` | All references resolved successfully |
| `Validated` | `False` | `UpstreamPoolNotFound` | Referenced pool does not exist |
| `Validated` | `False` | `CacheProfileNotFound` | Referenced cache profile does not exist |
