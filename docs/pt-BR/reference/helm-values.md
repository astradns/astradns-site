# Referencia de Valores do Helm

Referencia completa para `deploy/helm/astradns/values.yaml`.

## Operator

```yaml
operator:
  image:
    repository: astradns/operator    # Repositorio da imagem de container
    tag: ""                          # Tag (padrao: appVersion do Chart)
    pullPolicy: IfNotPresent
  replicas: 1                        # Numero de replicas do operator
  leaderElect: true                  # Habilitar eleicao de lider para HA
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi
  nodeSelector: {}
  tolerations: []
  affinity: {}
```

## Agent

```yaml
agent:
  image:
    repository: astradns/agent       # Repositorio da imagem de container
    tag: ""                          # Tag (padrao: appVersion do Chart)
    pullPolicy: IfNotPresent
  engineType: unbound                # Engine DNS: unbound, coredns, powerdns
  network:
    mode: hostPort                   # hostPort ou linkLocal
    linkLocalIP: 169.254.20.11       # IP link-local (quando mode=linkLocal)
  logMode: sampled                   # full, sampled, errors-only, off
  logSampleRate: "0.1"               # Taxa de amostragem para o modo sampled
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
  engineImages: {}                   # Sobrescritas de imagem por engine
  nodeSelector: {}
  tolerations: []
  affinity: {}
```

### Sobrescritas de Imagem do Engine

```yaml
agent:
  engineImages:
    coredns: "my-registry/coredns:1.12.1"
    powerdns: "my-registry/pdns-recursor:5.2"
```

## CRDs

```yaml
crds:
  install: true                      # Instalar CRDs com o chart
```

## Webhook

```yaml
webhook:
  enabled: false                     # Habilitar webhook de validacao
  certManager:
    issuerRef:
      name: ""                       # Nome do Issuer do cert-manager (obrigatorio quando habilitado)
      kind: ClusterIssuer            # Issuer ou ClusterIssuer
```

!!! warning "cert-manager necessario"
    O webhook requer cert-manager para provisionar certificados TLS. Certifique-se de que o cert-manager esta instalado e o issuer existe antes de habilitar.

## Observabilidade

```yaml
serviceMonitor:
  enabled: false                     # Criar ServiceMonitor para o Prometheus Operator

grafana:
  dashboards:
    enabled: false                   # Criar ConfigMap para descoberta via sidecar do Grafana
```

## Integracao com CoreDNS

```yaml
coredns:
  integration:
    enabled: false                   # Patch no CoreDNS para encaminhar consultas externas
```

Quando habilitado, um Job Kubernetes pos-instalacao:

1. Faz backup do ConfigMap atual do CoreDNS
2. Adiciona uma diretiva `forward` apontando zonas externas para o agent AstraDNS
3. Reinicia o deployment do CoreDNS para aplicar a alteracao

## Perfil de Producao

Um arquivo de valores pronto para producao e fornecido em `values-production.yaml`:

```bash
helm install astradns deploy/helm/astradns \
  -f deploy/helm/astradns/values-production.yaml \
  --set webhook.certManager.issuerRef.name=your-issuer
```

Isso habilita: modo de rede linkLocal, integracao com CoreDNS, aplicacao de webhook, ServiceMonitor e dashboards Grafana.
