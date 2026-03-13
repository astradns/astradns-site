# Referencia de Valores de Helm

Esta página documenta los knobs más relevantes de `deploy/helm/astradns/values.yaml` con foco operativo.

!!! info "Alcance"
    Esta es una referencia práctica, no un volcado completo y literal del chart. Resalta valores que cambian la forma del despliegue, el enrutamiento DNS, la postura de seguridad y la observabilidad.

---

## Modelo de Gestion de Imagenes

Helm administra automaticamente repositorio y version de imagen. El usuario solo elige `agent.engineType`.

- Imagen del operator: `ghcr.io/astradns/astradns-operator:v<appVersion>`
- Imagen del agent: `ghcr.io/astradns/astradns-agent:v<appVersion>-<engine>`

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
    automountServiceAccountToken: true

agent:
  serviceAccount:
    automountServiceAccountToken: false
```

!!! note "Politica de token"
    El operator necesita acceso a la API y mantiene token mount habilitado. El token del agent permanece deshabilitado por defecto, salvo que su flujo realmente requiera acceso a la API Kubernetes desde el pod.

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
