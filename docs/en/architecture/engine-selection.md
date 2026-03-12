# Engine Selection

AstraDNS supports three DNS engines. The engine is selected at deploy time via the `ASTRADNS_ENGINE_TYPE` environment variable (or `agent.engineType` Helm value).

## Comparison

| Feature | Unbound | CoreDNS | PowerDNS Recursor |
|---------|---------|---------|-------------------|
| **Type** | Recursive resolver | Authoritative + forwarding | Recursive resolver |
| **Language** | C | Go | C++ |
| **Memory footprint** | ~20-50 MB | ~30-60 MB | ~40-80 MB |
| **Cache performance** | Excellent | Good | Good |
| **DNSSEC validation** | Built-in | Plugin-based | Built-in |
| **Config reload** | `unbound-control reload` | Auto-reload plugin | `rec_control reload-zones` |
| **Maturity** | 20+ years | 10+ years | 15+ years |
| **Default** | Yes | No | No |

## When to Use Each

### Unbound (default)

Best for most deployments. Unbound offers:

- Best-in-class caching with aggressive NSEC optimization
- Lowest memory footprint
- Simplest configuration (single file)
- Battle-tested in production for 20+ years

```yaml
# values.yaml
agent:
  engineType: unbound
```

### CoreDNS

Choose CoreDNS when:

- Your team already operates CoreDNS and wants a single DNS technology
- You need custom plugins (e.g., service discovery, custom middleware)
- You want Go-native observability integration

```yaml
# values.yaml
agent:
  engineType: coredns
  engineImages:
    coredns: coredns/coredns:1.12.1
```

### PowerDNS Recursor

Choose PowerDNS when:

- You need Lua scripting for response policy zones (RPZ)
- Your organization standardizes on PowerDNS
- You need advanced EDNS Client Subnet support

```yaml
# values.yaml
agent:
  engineType: powerdns
  engineImages:
    powerdns: powerdns/pdns-recursor:5.2
```

## Engine Images

By default, the agent container ships with Unbound pre-installed. For other engines, you must provide a custom image or specify the engine image:

```yaml
agent:
  engineImages:
    coredns: "my-registry/coredns:1.12.1"
    powerdns: "my-registry/pdns-recursor:5.2"
```

!!! warning
    The default agent image only includes Unbound binaries. Using `engineType: coredns` or `engineType: powerdns` without providing the engine image or a custom agent image will fail at startup.
