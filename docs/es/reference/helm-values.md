# Referencia de Valores de Helm

Esta página documenta los knobs más relevantes de `deploy/helm/astradns/values.yaml` con foco operativo.

!!! info "Alcance"
    Esta es una referencia práctica, no un volcado completo y literal del chart. Resalta valores que cambian la forma del despliegue, el enrutamiento DNS, la postura de seguridad y la observabilidad.

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

  # Overrides opcionales por motor
  engineImages:
    unbound: { repository: "", tag: "" }
    coredns: { repository: "", tag: "" }
    powerdns: { repository: "", tag: "" }
    bind: { repository: "", tag: "" }
```

---

## Topología del Agent

```yaml
agent:
  topology:
    profile: node-local        # node-local | central

  network:
    mode: hostPort             # hostPort | linkLocal
    linkLocalIP: 169.254.20.11

  deployment:                  # usado en profile=central
    replicas: 3
    strategy:
      type: RollingUpdate
    topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule

  dnsService:                  # usado en profile=central
    type: ClusterIP
    clusterIP: ""             # opcional; IP fija recomendada en producción
    port: 53
    sessionAffinity: ClientIP
    sessionAffinityTimeoutSeconds: 1800
```

### Reglas importantes

- `profile=central` no puede usar `network.mode=linkLocal`.
- `profile=node-local` con patch de CoreDNS requiere `network.mode=linkLocal`.
- `central` renderiza `Deployment + Service`.
- `node-local` renderiza `DaemonSet`.

---

## Comportamiento del patch de CoreDNS

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

### Cómo se selecciona el target de forward

- `node-local`: usa `forwardTarget` directamente.
- `central`: ignora `forwardTarget` y usa:
  1. `agent.dnsService.clusterIP` cuando está definido;
  2. descubrimiento en runtime del `clusterIP` del Service cuando está vacío.

---

## Performance de proxy/runtime

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

## Observabilidad y diagnóstico

```yaml
agent:
  metrics:
    bearerToken: ""           # protección opcional de /metrics

  diagnostics:
    enabled: false
    targets: ""               # ej: s3.us-west-004.backblazeb2.com
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

## Defaults de seguridad

```yaml
operator:
  serviceAccount:
    automountServiceAccountToken: false

agent:
  serviceAccount:
    automountServiceAccountToken: false
```

!!! note "Cuándo habilitar token mount"
    Define `automountServiceAccountToken=true` solo cuando el workload realmente necesita acceso a la API de Kubernetes en runtime.

---

## Ejemplo de instalación en producción

```bash
helm upgrade --install astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --version 0.2.5 \
  -n astradns-system --create-namespace \
  -f values-production.yaml \
  --set webhook.certManager.issuerRef.name=<cluster-issuer>
```

---

## Relacionado

- [Perfiles de Topología](../guides/topology-profiles.md)
- [Integración con CoreDNS](../guides/coredns-integration.md)
- [Runbook](../operations/runbook.md)
