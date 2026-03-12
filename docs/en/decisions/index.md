# Architecture Decision Records

Key technical decisions behind AstraDNS, documented for transparency and future reference.

## ADR Index

| ADR | Title | Status | Summary |
|-----|-------|--------|---------|
| [001](adr-001.md) | Data Path Interception | Accepted | NodeLocal DNS pattern with CoreDNS forwarding |
| [002](adr-002.md) | Policy Scope in MVP | Accepted | Namespace-only scope; workload/domain scopes deferred |
| [003](adr-003.md) | Metrics Model | Accepted | Node-level metrics only in MVP; namespace requires direct datapath |
| [004](adr-004.md) | Failover Strategy | Accepted | Engine-native failover in MVP; agent circuit breaker post-MVP |
| [005](adr-005.md) | Naming and Positioning | Accepted | "AstraDNS" with `dns.astradns.com` API group |
| [006](adr-006.md) | Namespace Enrichment | Accepted | Accept limitation in MVP; pod informer post-MVP |
| [007](adr-007.md) | Query Logging Strategy | Accepted | Agent-level structured logging with sampling |
| [008](adr-008.md) | Multi-Engine Strategy | Accepted | Interface from day 1; Unbound only in MVP |
| [009](adr-009.md) | Agent Topology Profiles | Accepted | Two profiles (node-local, central) for cost vs latency trade-off |

The source ADR files live in `astradns-docs/adrs/` and are mirrored here as versioned site pages.
