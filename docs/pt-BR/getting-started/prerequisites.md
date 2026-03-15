# Pre-requisitos

## Requisitos do Cluster

| Requisito | Minimo | Recomendado |
|-----------|--------|-------------|
| Kubernetes | v1.30+ | v1.32+ |
| Helm | v3.12+ | v3.16+ |
| Container runtime | Qualquer compativel com OCI | containerd |
| CNI | Qualquer | Calico, Cilium ou Flannel |

## Dependencias Opcionais

| Componente | Necessario Para | Instalacao |
|------------|----------------|------------|
| cert-manager | TLS do webhook de validacao | [cert-manager.io](https://cert-manager.io/docs/installation/) |
| Prometheus Operator | Coleta via ServiceMonitor | [prometheus-operator](https://github.com/prometheus-operator/prometheus-operator) |
| Grafana | Dashboards DNS | Qualquer Grafana com provisionamento via sidecar |

## Requisitos de Recursos

### Operator (Deployment)

O operator e leve -- ele observa CRDs e escreve ConfigMaps.

| Recurso | Request | Limit |
|---------|---------|-------|
| CPU | 100m | 200m |
| Memoria | 128Mi | 256Mi |

### Agent (DaemonSet)

O agent executa em cada no e lida com o trafego DNS.

| Recurso | Request | Limit |
|---------|---------|-------|
| CPU | 100m | 500m |
| Memoria | 128Mi | 512Mi |

!!! tip "Dimensionamento para nos de alto trafego"
    Nos que atendem >5.000 QPS podem precisar de limites de CPU mais altos. Monitore a taxa de `astradns_queries_total` para identificar nos com alta demanda.

## Requisitos de Rede

### Modo hostPort (padrao)

O agent se conecta a porta `5353` em cada no via `hostPort`. Nenhuma configuracao especial de rede e necessaria.

### Modo linkLocal (recomendado para producao)

Por padrao, o agent se conecta ao endereco link-local `169.254.20.11:5353` em cada no. Este modo requer:

- `hostNetwork: true` no DaemonSet do agent (configurado automaticamente pelo Helm)
- CoreDNS configurado para encaminhar consultas externas para `169.254.20.11:5353` (automatizado via `clusterDNS.forwardExternalToAstraDNS.enabled=true`)
- Se voce customizar `agent.network.linkLocalIP`, atualize `clusterDNS.forwardExternalToAstraDNS.forwardTarget` para `<linkLocalIP>:5353`

## RBAC

O AstraDNS cria os seguintes recursos RBAC com escopo de cluster:

- **ClusterRole** para o operator (ler CRDs, escrever ConfigMaps, criar eventos)
- **ClusterRoleBinding** vinculando a ServiceAccount do operator
- **ServiceAccount** por componente (operator, agent)

Nenhum privilegio de cluster-admin e necessario.
