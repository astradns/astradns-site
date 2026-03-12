# Helm Values Reference

This page documents the most relevant knobs from `deploy/helm/astradns/values.yaml` with an operational focus.

!!! info "Scope"
    This is a practical reference, not a full verbatim dump of the chart. It highlights values that change deployment shape, DNS routing behavior, security posture, and observability.

---

## Operator

```yaml
operator:
  image:
    repository: ghcr.io/astradns/astradns-operator
    tag: ""                  # default: v<appVersion>
    pullPolicy: IfNotPresent

  replicas: 1
  leaderElect: true
  leaderElection:
    id: 60acac32.astradns.com

  serviceAccount:
    automountServiceAccountToken: false

  lifecycle:
    preStop:
      enabled: true
      sleepSeconds: 5
```

---

## Agent (general)

```yaml
agent:
  image:
    repository: ghcr.io/astradns/astradns-agent
    tag: v0.2.0
    pullPolicy: IfNotPresent

  engineType: unbound          # unbound | coredns | powerdns | bind

  # Optional per-engine overrides
  engineImages:
    unbound: { repository: "", tag: "" }
    coredns: { repository: "", tag: "" }
    powerdns: { repository: "", tag: "" }
    bind: { repository: "", tag: "" }
```

---

## Agent topology

```yaml
agent:
  topology:
    profile: node-local        # node-local | central

  network:
    mode: hostPort             # hostPort | linkLocal
    linkLocalIP: 169.254.20.11

  deployment:                  # used in profile=central
    replicas: 3
    strategy:
      type: RollingUpdate
    topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule

  dnsService:                  # used in profile=central
    type: ClusterIP
    clusterIP: ""             # optional; fixed Service IP recommended in production
    port: 53
    sessionAffinity: ClientIP
    sessionAffinityTimeoutSeconds: 1800
```

### Important rules

- `profile=central` cannot use `network.mode=linkLocal`.
- `profile=node-local` with CoreDNS patching requires `network.mode=linkLocal`.
- `central` renders `Deployment + Service`.
- `node-local` renders `DaemonSet`.

---

## CoreDNS patch behavior

```yaml
clusterDNS:
  provider: coredns
  forwardExternalToAstraDNS:
    enabled: false
    namespace: kube-system
    configMapName: coredns
    rolloutDeployment: coredns
    forwardTarget: 169.254.20.11:5353
    fallbackUpstream: /etc/resolv.conf
```

### How the forward target is selected

- `node-local`: uses `forwardTarget` directly.
- `central`: ignores `forwardTarget` and uses:
  1. `agent.dnsService.clusterIP` when set;
  2. runtime Service `clusterIP` discovery when empty.

---

## Proxy/runtime performance

```yaml
agent:
  proxyTimeout: 2s
  proxyCacheMaxEntries: 10000
  proxyCacheDefaultTTL: 30s
  engineConnectionPoolSize: 64
  configWatchDebounce: 1s
  engineRecoveryInterval: 5s
  componentErrorBuffer: 5
```

---

## Observability and diagnostics

```yaml
agent:
  metrics:
    bearerToken: ""           # optional /metrics protection

  diagnostics:
    enabled: false
    targets: ""               # e.g. s3.us-west-004.backblazeb2.com
    interval: 1m
    timeout: 3s

  tracing:
    enabled: false
    endpoint: localhost:4318
    insecure: true
    sampleRatio: "0.1"
    serviceName: astradns-agent

  logMode: sampled
  logSampleRate: "0.1"
```

---

## Security defaults

```yaml
operator:
  serviceAccount:
    automountServiceAccountToken: false

agent:
  serviceAccount:
    automountServiceAccountToken: false
```

!!! note "When to enable token mount"
    Set `automountServiceAccountToken=true` only when the workload actually needs Kubernetes API access at runtime.

---

## Production install example

```bash
helm upgrade --install astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --version 0.2.5 \
  -n astradns-system --create-namespace \
  -f values-production.yaml \
  --set webhook.certManager.issuerRef.name=<cluster-issuer>
```

---

## Related

- [Topology Profiles](../guides/topology-profiles.md)
- [CoreDNS Integration](../guides/coredns-integration.md)
- [Runbook](../operations/runbook.md)
