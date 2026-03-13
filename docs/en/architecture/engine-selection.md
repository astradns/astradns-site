# Engine Selection

AstraDNS supports four DNS engines. You choose the engine with a single Helm value: `agent.engineType`.

## Supported Engines

| Engine | Typical fit | Notes |
|---|---|---|
| `unbound` | Default for most clusters | Strong recursive cache behavior and simple operations |
| `coredns` | Teams already standardized on CoreDNS | Operational consistency with existing CoreDNS expertise |
| `powerdns` | PowerDNS-centered environments | Useful when your DNS estate already uses PowerDNS tooling |
| `bind` | BIND-centered environments | Useful for teams with existing BIND operational practices |

## Image Policy (Helm)

The chart manages image repository and version automatically:

- Operator image: `ghcr.io/astradns/astradns-operator:v<appVersion>`
- Agent image: `ghcr.io/astradns/astradns-agent:v<appVersion>-<engine>`

This keeps operator + agent + docs aligned to the same release line. Users do not need to pick image tags.

## How to Choose an Engine

```yaml
agent:
  engineType: unbound # unbound | coredns | powerdns | bind
```

### Practical guidance

- Start with `unbound` unless you have a clear platform standard for another engine.
- Use `coredns`, `powerdns`, or `bind` when your operations team already runs that stack and has existing tooling/playbooks.
- Keep one engine per environment at a time; switch only with a controlled rollout window.

## Verify the Selected Engine

```bash
# Engine type propagated to operator
kubectl -n astradns-system get deploy astradns-operator \
  -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="ASTRADNS_ENGINE_TYPE")].value}'

# Agent image variant selected by Helm
kubectl -n astradns-system get ds astradns-agent \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```
