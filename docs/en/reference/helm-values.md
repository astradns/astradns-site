# Helm Values Reference

This page documents the most relevant knobs from the AstraDNS Helm chart values with an operational focus.

!!! info "Scope"
    This is a practical reference, not a full verbatim dump of the chart. It highlights values that change deployment shape, DNS routing behavior, security posture, and observability.

---

## Image Management Model

Helm manages image repository and version automatically. Users only choose `agent.engineType`.

- Operator image: `ghcr.io/astradns/astradns-operator:v<appVersion>`
- Agent image: `ghcr.io/astradns/astradns-agent:v<appVersion>-<engine>`

---

## Operator

```yaml
operator:
  imagePullPolicy: IfNotPresent

  replicas: 1
  leaderElect: true
  leaderElection:
    id: 60acac32.astradns.com

  serviceAccount:
    automountServiceAccountToken: true

  lifecycle:
    preStop:
      enabled: true
      sleepSeconds: 5
```

---

## Agent (general)

```yaml
agent:
  imagePullPolicy: IfNotPresent

  engineType: unbound          # unbound | coredns | powerdns | bind
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

!!! note "Link-local default, not a fixed contract"
    `169.254.20.11` is the chart default for `agent.network.linkLocalIP`, not a universal hardcoded value.
    If you change it, keep `clusterDNS.forwardExternalToAstraDNS.forwardTarget` aligned (`<linkLocalIP>:5353`) in `node-local` profile.

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
    forwardTarget: 169.254.20.11:5353  # node-local default; keep aligned with linkLocalIP
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
    automountServiceAccountToken: true

agent:
  serviceAccount:
    automountServiceAccountToken: false
```

!!! note "Token-mount policy"
    The operator requires API access and keeps token mount enabled. Agent token mount stays disabled by default unless your runtime workflow requires Kubernetes API access from the agent pod.

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
