# Architecture Decision Records

Key technical decisions behind AstraDNS, documented for transparency and future reference.

## ADR Index

| ADR | Title | Status | Summary |
|-----|-------|--------|---------|
| [001](#adr-001-data-path-interception) | Data Path Interception | Accepted | NodeLocal DNS pattern with CoreDNS forwarding |
| [002](#adr-002-policy-scope-in-mvp) | Policy Scope in MVP | Accepted | Namespace-only scope; workload/domain scopes deferred |
| [003](#adr-003-metrics-model) | Metrics Model | Accepted | Node-level metrics only in MVP; namespace requires direct datapath |
| [004](#adr-004-failover-strategy) | Failover Strategy | Accepted | Engine-native failover in MVP; agent circuit breaker post-MVP |
| [005](#adr-005-naming-and-positioning) | Naming and Positioning | Accepted | "AstraDNS" with `dns.astradns.com` API group |
| [006](#adr-006-namespace-enrichment) | Namespace Enrichment | Accepted | Accept limitation in MVP; pod informer post-MVP |
| [007](#adr-007-query-logging-strategy) | Query Logging Strategy | Accepted | Agent-level structured logging with sampling |
| [008](#adr-008-multi-engine-strategy) | Multi-Engine Strategy | Accepted | Interface from day 1; Unbound only in MVP |

---

## ADR-001: Data Path Interception

**Status:** Accepted

**Decision:** Use the NodeLocal DNS pattern — the AstraDNS Agent binds to a link-local address (`169.254.20.11`) on each node, and CoreDNS forwards external queries to it.

**Options evaluated:**

| Option | Pros | Cons |
|--------|------|------|
| iptables/nftables interception | Transparent, no CoreDNS changes | Fragile, CNI conflicts, hard to debug |
| Sidecar injection | Per-pod isolation | High overhead, mutating webhook complexity |
| Modify CoreDNS ConfigMap | Simple, proven pattern | Requires post-install job |
| **NodeLocal DNS (chosen)** | **Proven, per-node isolation, low risk** | **Requires link-local setup** |

**Consequences:** Each node has its own cache and resolver. A failure on one node does not affect others.

---

## ADR-002: Policy Scope in MVP

**Status:** Accepted

**Decision:** Support namespace-only scope in MVP. Workload-level and domain-level policy scopes are deferred.

**Key insight:** Namespace cannot be resolved from source IP in the CoreDNS forwarding data path. CNIs allocate IPs per-node, not per-namespace. True namespace-level enforcement requires the direct data path (pods query the agent directly).

**Consequences:** The `ExternalDNSPolicy` CRD is API-ready but enforcement is deferred to a future release that implements the direct data path.

---

## ADR-003: Metrics Model

**Status:** Accepted

**Decision:** MVP metrics are node-level only. Namespace-level metrics require the direct data path.

**Cardinality considerations:**

- `domain` labels are dangerous (unbounded cardinality)
- `pod` labels are risky (thousands of series)
- `node` labels are safe (bounded by cluster size)

**MVP metrics:** queries total, cache hits/misses, upstream latency/health/failures, SERVFAIL/NXDOMAIN counts, agent lifecycle.

---

## ADR-004: Failover Strategy

**Status:** Accepted

**Decision:** Two-phase approach:

1. **MVP:** Rely on engine-native failover (Unbound handles upstream failover natively). The agent adds observability via health checks and metrics.
2. **Post-MVP:** Agent-level circuit breaker with CLOSED / OPEN / HALF_OPEN states.

---

## ADR-005: Naming and Positioning

**Status:** Accepted

**Decision:** Name the project "AstraDNS" (navigation metaphor). API group: `dns.astradns.com`.

**Positioning vs related projects:**

| Project | Purpose | Overlap |
|---------|---------|---------|
| ExternalDNS | Publish DNS records to providers | None — different problem |
| CoreDNS | Cluster DNS server | Complementary — AstraDNS sits behind CoreDNS |
| node-local-dns | DNS caching addon | Similar pattern — AstraDNS adds observability and policy |

---

## ADR-006: Namespace Enrichment

**Status:** Accepted

**Decision:** Accept the limitation in MVP (namespace = "unknown" in metrics and logs). Post-MVP: pod informer with IP-to-namespace lookup table.

**Post-MVP design:** The agent maintains an in-memory table mapping pod IPs to namespaces, populated via a Kubernetes pod informer. Estimated memory: ~100 bytes per pod, so 1,000 pods = ~100KB.

---

## ADR-007: Query Logging Strategy

**Status:** Accepted

**Decision:** Agent-level structured JSON logging to stdout. Users integrate with their existing log pipeline (Loki, ELK, CloudWatch).

**Log modes:**

| Mode | Behavior |
|------|----------|
| `full` | Log every query |
| `sampled` | Log N% of successful queries, always log errors |
| `errors-only` | Log only NXDOMAIN and SERVFAIL |
| `off` | Disable query logging |

---

## ADR-008: Multi-Engine Strategy

**Status:** Accepted

**Decision:** Design the `Engine` interface from day 1. Implement only Unbound in MVP, then add CoreDNS and PowerDNS.

**Phases:**

1. MVP: Unbound only
2. Add CoreDNS + engine selection via Helm value
3. Add PowerDNS

**Consequence:** The agent uses the Engine interface from the start, making it trivial to add new engines without changing the proxy, metrics, or logging layers.
