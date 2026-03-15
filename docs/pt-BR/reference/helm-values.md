# Referência de Valores Helm

Esta página documenta os valores mais importantes do chart Helm do AstraDNS com foco operacional.

!!! info "Escopo"
    Nem todos os campos do chart são reproduzidos aqui. Esta referência cobre os knobs que mudam comportamento de deploy, tráfego DNS, segurança e observabilidade.

---

## Modelo de Gerenciamento de Imagens

O Helm gerencia automaticamente repositorio e versao das imagens. O usuario escolhe apenas `agent.engineType`.

- Imagem do operator: `ghcr.io/astradns/astradns-operator:v<appVersion>`
- Imagem do agent: `ghcr.io/astradns/astradns-agent:v<appVersion>-<engine>`

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

## Agent (geral)

```yaml
agent:
  imagePullPolicy: IfNotPresent

  engineType: unbound          # unbound | coredns | powerdns | bind
```

---

## Topologia do Agent

```yaml
agent:
  topology:
    profile: node-local        # node-local | central

  network:
    mode: hostPort             # hostPort | linkLocal
    linkLocalIP: 169.254.20.11

  deployment:                  # usado em profile=central
    replicas: 3
    strategy:
      type: RollingUpdate
    topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule

  dnsService:                  # usado em profile=central
    type: ClusterIP
    clusterIP: ""             # opcional; IP fixo recomendado em produção
    port: 53
    sessionAffinity: ClientIP
    sessionAffinityTimeoutSeconds: 1800
```

### Regras importantes

- `profile=central` não aceita `network.mode=linkLocal`.
- `profile=node-local` com patch do CoreDNS exige `network.mode=linkLocal`.
- Em `central`, o chart cria `Deployment + Service`.
- Em `node-local`, o chart cria `DaemonSet`.

---

## Comportamento do CoreDNS patch

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

### Como o target é escolhido

- Em `node-local`: usa `forwardTarget` explicitamente.
- Em `central`: ignora `forwardTarget` e usa:
  1. `agent.dnsService.clusterIP` quando definido;
  2. descoberta automática do `clusterIP` do Service quando vazio.

---

## Runtime e performance do proxy

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

## Observabilidade e diagnóstico

```yaml
agent:
  metrics:
    bearerToken: ""           # opcional para proteger /metrics

  diagnostics:
    enabled: false
    targets: ""               # ex: s3.us-west-004.backblazeb2.com
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

## Segurança

```yaml
operator:
  serviceAccount:
    automountServiceAccountToken: true

agent:
  serviceAccount:
    automountServiceAccountToken: false
```

!!! note "Politica de token"
    O operator precisa de acesso a API e mantém token mount habilitado. O token do agent continua desabilitado por padrao, exceto quando seu fluxo realmente exigir acesso a API Kubernetes dentro do pod.

---

## Produção (perfil recomendado)

```bash
helm upgrade --install astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --version 0.2.5 \
  -n astradns-system --create-namespace \
  -f values-production.yaml \
  --set webhook.certManager.issuerRef.name=<cluster-issuer>
```

---

## Relacionados

- [Guia: Perfis de Topologia](../guides/topology-profiles.md)
- [Guia: Integração CoreDNS](../guides/coredns-integration.md)
- [Runbook](../operations/runbook.md)
