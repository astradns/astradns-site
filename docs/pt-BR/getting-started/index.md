# Primeiros Passos

O AstraDNS e instalado como um chart Helm e nao requer alteracoes no codigo da sua aplicacao. Este guia orienta voce em uma configuracao minima em qualquer cluster Kubernetes.

## O Que Voce Obtem

Apos a instalacao, cada no do seu cluster executa um AstraDNS Agent que:

- **Encaminha** consultas DNS externas atraves de um resolver gerenciado (Unbound por padrao)
- **Armazena em cache** respostas com TTLs configuraveis
- **Registra** cada consulta como JSON estruturado
- **Exporta** metricas Prometheus sobre acertos de cache, latencia, falhas e saude dos upstreams
- **Verifica a saude** dos resolvers upstream e reporta o status via condicoes do CRD

## Instalacao em 5 Minutos

### 1. Adicione o chart Helm

```bash
helm install astradns deploy/helm/astradns \
  --namespace astradns-system \
  --create-namespace
```

### 2. Crie um pool de upstreams

```yaml
apiVersion: dns.astradns.com/v1alpha1
kind: DNSUpstreamPool
metadata:
  name: default
  namespace: astradns-system
spec:
  upstreams:
    - address: "1.1.1.1"
      port: 53
    - address: "8.8.8.8"
      port: 53
```

```bash
kubectl apply -f pool.yaml
```

### 3. Verifique

```bash
# Verifique se o operator esta em execucao
kubectl get pods -n astradns-system -l app.kubernetes.io/component=operator

# Verifique se o agent esta em execucao nos nos
kubectl get pods -n astradns-system -l app.kubernetes.io/component=agent

# Verifique o status do pool
kubectl get dnsupstreampools -n astradns-system
```

O pool deve exibir `Ready=True` em poucos segundos.

### 4. Teste a resolucao DNS

```bash
kubectl run dns-test --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup example.com
```

## Proximos Passos

- [Pre-requisitos](prerequisites.md) para clusters de producao
- [Instalacao](installation.md) com valores de producao
- [Configuracao do Helm](../guides/helm-configuration.md) referencia
- [Integracao com CoreDNS](../guides/coredns-integration.md) para DNS local no no
