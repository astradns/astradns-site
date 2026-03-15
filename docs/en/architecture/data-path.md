# Data Path

Understanding how DNS queries flow through AstraDNS is critical for debugging, capacity planning, and security analysis.

## Query Flow

```mermaid
sequenceDiagram
    participant Pod
    participant CoreDNS
    participant Proxy as Agent Proxy
    participant Engine as DNS Engine
    participant Upstream as Upstream Resolver

    Pod->>CoreDNS: A api.stripe.com?
    Note over CoreDNS: Internal zone? No.
    CoreDNS->>Proxy: Forward to 169.254.20.11:5353
    Proxy->>Proxy: Rate limit check
    Proxy->>Proxy: Domain filter check
    alt Denied by filter
        Proxy-->>CoreDNS: REFUSED / NXDOMAIN
    end
    alt Proxy cache hit
        Proxy-->>CoreDNS: Cached response
    end
    Proxy->>Engine: Forward to 127.0.0.1:5354
    alt Engine responds
        Engine-->>Proxy: Response (from engine cache or upstream)
    else Engine unreachable
        Proxy->>Proxy: Try stale cache (expired entries)
        alt Stale cache hit
            Proxy-->>CoreDNS: Stale response (TTL=1)
        end
        Proxy->>Upstream: Direct fallback query
        alt Upstream responds
            Upstream-->>Proxy: Response
        else All failed
            Proxy-->>CoreDNS: SERVFAIL
        end
    end
    Proxy->>Proxy: Record latency, emit event
    Proxy-->>CoreDNS: DNS response
    CoreDNS-->>Pod: DNS response
```

## Network Modes

### hostPort (default)

The agent binds to port `5353` on the node's primary IP via Kubernetes `hostPort`.

```
Pod → CoreDNS → Node:5353 (agent) → 127.0.0.1:5354 (engine) → upstream
```

- Simple to set up
- No host network required
- CoreDNS must be configured to forward to `<nodeIP>:5353`

### linkLocal (recommended)

By default, the agent binds to the link-local address `169.254.20.11:5353` using `hostNetwork: true`.

```
Pod → CoreDNS → <linkLocalIP>:5353 (agent) → 127.0.0.1:5354 (engine) → upstream
```

- Stable link-local address and port for every node
- Consistent configured address across all nodes
- CoreDNS forwards to a single address regardless of node IP
- Follows the established NodeLocal DNS Cache pattern

!!! tip "Production recommendation"
    Use linkLocal mode with CoreDNS integration enabled. This is the most reliable and widely-tested configuration.

## Cache Behavior

Each node maintains its own cache. There is no cross-node cache sharing.

| Property | Controlled By |
|----------|--------------|
| Maximum entries | `DNSCacheProfile.spec.maxEntries` |
| Minimum positive TTL | `DNSCacheProfile.spec.positiveTtl.minSeconds` |
| Maximum positive TTL | `DNSCacheProfile.spec.positiveTtl.maxSeconds` |
| Negative TTL | `DNSCacheProfile.spec.negativeTtl.seconds` |
| Prefetch | `DNSCacheProfile.spec.prefetch.enabled` |
| Prefetch threshold | `DNSCacheProfile.spec.prefetch.threshold` |

!!! info "Per-node cache isolation"
    Cache is isolated per node. A query cached on Node A is not available on Node B. This is a deliberate design choice for fault isolation — a poisoned cache on one node does not propagate to others.

## Failure Modes

| Failure | Behavior | Recovery |
|---------|----------|----------|
| Agent pod crashes | CoreDNS fails over to `/etc/resolv.conf` (sequential policy) | Agent restart via DaemonSet/Deployment (~seconds) |
| Engine subprocess dies | Proxy serves stale cache or resolves via upstream directly | Engine supervisor restarts within 5s |
| Engine down + no cache | Proxy queries upstreams directly, bypassing engine | Transparent — no client impact |
| Engine down + all upstreams down | SERVFAIL for non-cached queries | Automatic when engine or upstream recovers |
| Upstream unreachable | Health checker marks unhealthy, SERVFAIL returned | Automatic when upstream recovers |
| ConfigMap invalid | Reload fails, previous config retained | Fix CRD, operator re-renders |
| Operator down | No config changes processed, existing config continues | Operator restarts via Deployment |

### Resilience Chain

When the DNS engine is unreachable, the proxy follows this fallback chain before returning SERVFAIL:

```
1. Proxy cache (fresh)     → instant response
2. Engine                   → normal path
3. Stale cache (expired)   → response with TTL=1 (up to 5 min after expiry)
4. Direct upstream query   → bypasses engine, resolves via configured upstreams
5. SERVFAIL                → all paths exhausted
```

Domain filtering and metrics continue to work in all degraded modes. Only engine-level features (DNSSEC validation, engine cache prefetch) are unavailable during fallback.

## Protocol Support

| Feature | Status |
|---------|--------|
| UDP queries | Supported |
| TCP queries | Supported |
| EDNS | Passthrough |
| DNSSEC | Passthrough (engine validates if configured) |
| DNS-over-TLS | Not supported (planned) |
| DNS-over-HTTPS | Not supported (planned) |
