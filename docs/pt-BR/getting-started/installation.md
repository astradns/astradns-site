# Instalacao

## Instalacao via Helm

### Instalacao Basica

```bash
helm upgrade --install astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system \
  --create-namespace \
  --set agent.engineType=unbound
```

### Instalacao para Producao

Use o perfil de valores de producao para uma implantacao robusta:

```bash
helm upgrade --install astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system \
  --create-namespace \
  -f values-production.yaml \
  --set webhook.certManager.issuerRef.name=your-cluster-issuer
```

Isso habilita:

- **Caminho de dados link-local** -- o agent se conecta a `169.254.20.11` para DNS verdadeiramente local no no
- **Integracao com CoreDNS** -- patch automatico do DNS do cluster para encaminhar consultas externas
- **Webhook de validacao** -- impoe um unico `DNSUpstreamPool` por namespace
- **ServiceMonitor** -- coleta Prometheus para metricas do operator e do agent
- **Dashboard Grafana** -- dashboard pre-construido de observabilidade DNS

### Verificar a Instalacao

```bash
# Todos os pods devem estar Running
kubectl get pods -n astradns-system

# Os CRDs devem estar registrados
kubectl get crds | grep astradns

# Saida esperada:
# dnscacheprofiles.dns.astradns.com
# dnsupstreampools.dns.astradns.com
# externaldnspolicies.dns.astradns.com
```

## Configuracao Pos-Instalacao

### Criar um Pool de Upstreams

Cada namespace que utiliza o AstraDNS precisa de um `DNSUpstreamPool`:

```yaml
apiVersion: dns.astradns.com/v1alpha1
kind: DNSUpstreamPool
metadata:
  name: production
  namespace: astradns-system
spec:
  upstreams:
    - address: "1.1.1.1"
    - address: "8.8.8.8"
    - address: "9.9.9.9"
  healthCheck:
    enabled: true
    intervalSeconds: 30
    timeoutSeconds: 5
    failureThreshold: 3
  loadBalancing:
    strategy: round-robin
```

### Configurar Cache (Opcional)

```yaml
apiVersion: dns.astradns.com/v1alpha1
kind: DNSCacheProfile
metadata:
  name: default
  namespace: astradns-system
spec:
  maxEntries: 50000
  positiveTtl:
    minSeconds: 60
    maxSeconds: 300
  negativeTtl:
    seconds: 30
  prefetch:
    enabled: true
    threshold: 5
```

!!! note
    O perfil de cache com o nome `default` e automaticamente utilizado pelo operator. Nenhuma referencia explicita e necessaria no pool de upstreams.

## Desinstalacao

```bash
# Remova os CRs primeiro
kubectl delete dnsupstreampools,dnscacheprofiles,externaldnspolicies \
  --all -n astradns-system

# Desinstale o chart
helm uninstall astradns -n astradns-system

# Remova os CRDs (opcional)
kubectl delete crds \
  dnsupstreampools.dns.astradns.com \
  dnscacheprofiles.dns.astradns.com \
  externaldnspolicies.dns.astradns.com

# Remova o namespace
kubectl delete namespace astradns-system
```

!!! warning
    Excluir os CRDs removera todos os recursos personalizados do AstraDNS em todos os namespaces. Esta acao e irreversivel.
