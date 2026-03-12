# Integracao com CoreDNS

O AstraDNS se integra com o CoreDNS do cluster para interceptar consultas DNS externas. Este guia explica como a integracao funciona e como configura-la.

## Como Funciona

Quando `coredns.integration.enabled=true`, o chart Helm cria um Job pos-instalacao que:

1. Cria RBAC (ServiceAccount, Role, RoleBinding) em `kube-system`
2. Le o ConfigMap atual do CoreDNS
3. Faz backup do Corefile original em um novo ConfigMap
4. Adiciona uma diretiva `forward` que roteia consultas nao pertencentes ao cluster para o AstraDNS
5. Reinicia o deployment do CoreDNS para aplicar a alteracao

### Antes da Integracao

```
.:53 {
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
    }
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

### Apos a Integracao

```
.:53 {
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
    }
    forward . 169.254.20.11           # <-- AstraDNS agent
    cache 30
    loop
    reload
    loadbalance
}
```

## Configuracao

### Habilitar via Helm

```yaml
agent:
  network:
    mode: linkLocal                   # Necessario para integracao com CoreDNS
    linkLocalIP: 169.254.20.11

coredns:
  integration:
    enabled: true
```

### Modo de Rede Necessario

A integracao com o CoreDNS requer o modo de rede `linkLocal`. O agent se conecta a `169.254.20.11:53` em cada no, fornecendo um endereco consistente para o encaminhamento do CoreDNS.

!!! warning
    O uso do modo `hostPort` com a integracao CoreDNS nao e suportado porque o CoreDNS precisaria saber o endereco IP de cada no, que varia ao longo do cluster.

## Verificacao

Apos a instalacao, verifique a integracao:

```bash
# Verifique se o ConfigMap do CoreDNS foi alterado
kubectl get configmap coredns -n kube-system -o yaml | grep forward

# Verifique se o backup existe
kubectl get configmap coredns-backup-astradns -n kube-system

# Teste a resolucao DNS atraves do AstraDNS
kubectl run dns-test --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup example.com
```

Verifique as metricas do agent para confirmar que as consultas estao fluindo:

```bash
kubectl port-forward -n astradns-system ds/astradns-agent 9153:9153
curl -s http://localhost:9153/metrics | grep astradns_queries_total
```

## Reverter

Para reverter manualmente a integracao com o CoreDNS:

```bash
# Restaurar a partir do backup
kubectl get configmap coredns-backup-astradns -n kube-system \
  -o jsonpath='{.data.Corefile}' | \
  kubectl create configmap coredns -n kube-system \
  --from-file=Corefile=/dev/stdin --dry-run=client -o yaml | \
  kubectl apply -f -

# Reiniciar o CoreDNS
kubectl rollout restart deployment coredns -n kube-system
```

Ou simplesmente desinstale o AstraDNS -- o processo de `helm uninstall` nao reverte automaticamente as alteracoes do CoreDNS, entao a reversao manual pode ser necessaria.
