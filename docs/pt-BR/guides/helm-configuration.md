# Configuracao do Helm

## Instalacao

### A Partir do Chart OCI Oficial

```bash
helm upgrade --install astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system \
  --create-namespace
```

### Escolha do Engine DNS

A unica escolha de engine que o usuario precisa fazer e `agent.engineType`.

```bash
helm upgrade --install astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system \
  --create-namespace \
  --set agent.engineType=unbound
```

Valores suportados: `unbound`, `coredns`, `powerdns`, `bind`.

!!! info "Politica de imagens"
    O chart fixa automaticamente as imagens oficiais:
    - `ghcr.io/astradns/astradns-operator:v<appVersion>`
    - `ghcr.io/astradns/astradns-agent:v<appVersion>-<engine>`

### Com Valores Personalizados

```bash
helm upgrade --install astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system \
  --create-namespace \
  -f my-values.yaml
```

## Perfis de Topologia

### Node-Local (padrao)

```yaml
agent:
  topology:
    profile: node-local
  engineType: unbound
  network:
    mode: linkLocal

clusterDNS:
  forwardExternalToAstraDNS:
    enabled: true
```

### Central

```yaml
agent:
  topology:
    profile: central
  engineType: unbound
  deployment:
    replicas: 3
  dnsService:
    type: ClusterIP
    sessionAffinity: ClientIP
```

## Perfis Comuns

### Baseline de Producao

```yaml
operator:
  leaderElect: true

agent:
  engineType: unbound
  topology:
    profile: node-local
  network:
    mode: linkLocal
  logMode: sampled
  logSampleRate: "0.1"

clusterDNS:
  forwardExternalToAstraDNS:
    enabled: true

webhook:
  enabled: true

serviceMonitor:
  enabled: true

grafana:
  dashboards:
    enabled: true
```

### Cluster de Alto Trafego (Central)

```yaml
agent:
  engineType: unbound
  topology:
    profile: central
  deployment:
    replicas: 5
  resources:
    requests:
      cpu: 250m
      memory: 256Mi
    limits:
      cpu: "1"
      memory: "1Gi"
  logMode: sampled
  logSampleRate: "0.05"
```

## Validacao Pos-Instalacao

```bash
# 1) Operator e agent saudaveis
kubectl -n astradns-system get pods -l app.kubernetes.io/component=operator
kubectl -n astradns-system get pods -l app.kubernetes.io/component=agent

# 2) ConfigMap existe
kubectl -n astradns-system get configmap astradns-agent-config

# 3) Smoke test DNS
kubectl run dns-test --rm -it --restart=Never --image=busybox:1.37 -- nslookup example.com
```

## Atualizacao

```bash
helm upgrade astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system \
  -f my-values.yaml
```

!!! tip "Execute um dry run primeiro"
    Sempre visualize as alteracoes antes de aplica-las:
    ```bash
    helm upgrade astradns oci://ghcr.io/astradns/helm-charts/astradns \
      --namespace astradns-system \
      -f my-values.yaml \
      --dry-run --debug
    ```

## Renderizacao de Templates

Para inspecionar os manifests renderizados sem instalar:

```bash
helm template astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system \
  -f my-values.yaml
```

Consulte a [Referencia de Valores do Helm](../reference/helm-values.md) para todos os valores disponiveis.
