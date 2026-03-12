# Solucao de Problemas

## Diagnostico Rapido

```bash
# Verificacao de saude em uma unica linha
kubectl get pods,dnsupstreampools,dnscacheprofiles -n astradns-system
```

## Problemas Comuns

### "No CRDs to install; skipping" durante helm install

**Causa:** Os CRDs nao estao sendo instalados.

**Correcao:** Certifique-se de que `crds.install=true` esta nos seus valores:

```bash
helm install astradns deploy/helm/astradns --set crds.install=true
```

### Pod do agent em CrashLoopBackOff

**Diagnostico:**

```bash
kubectl logs -n astradns-system -l app.kubernetes.io/component=agent --previous
kubectl describe pod -n astradns-system -l app.kubernetes.io/component=agent
```

**Causas comuns:**

| Mensagem de log | Causa | Correcao |
|-----------------|-------|----------|
| `exec: "unbound": not found` | Tipo de engine incorreto para a imagem | Combine `agent.engineType` com o binario do engine na imagem |
| `listen tcp :5353: bind: address already in use` | Conflito de porta | Verifique se ha outros servicos na porta 5353 |
| `open /etc/astradns/config/config.json: no such file` | ConfigMap nao montado | Verifique se o ConfigMap existe e esta referenciado no DaemonSet |

### Pool exibe Ready=False, Reason=Superseded

**Causa:** Multiplos recursos `DNSUpstreamPool` existem no namespace. Apenas o pool mais antigo esta ativo.

**Correcao:**

```bash
# Visualizar todos os pools e seus status
kubectl get dnsupstreampools -n astradns-system -o yaml

# Excluir o pool suplantado
kubectl delete dnsupstreampool <superseded-name> -n astradns-system
```

### Metricas mostram zero acertos de cache

**Causas possiveis:**

1. **Cache nao aquecido:** Execute consultas repetidas para o mesmo dominio e verifique novamente
2. **Cache desabilitado:** Verifique se o `DNSCacheProfile` existe e `maxEntries > 0`
3. **Limitacao do proxy:** O proxy reporta o status do cache como "unknown" -- as metricas de cache vem do engine, nao do proxy

### Alta taxa de SERVFAIL

**Diagnostico:**

```bash
# Verificar saude dos upstreams
curl -s http://localhost:9153/metrics | grep astradns_upstream_healthy

# Verificar latencia dos upstreams
curl -s http://localhost:9153/metrics | grep astradns_upstream_latency
```

**Causas comuns:**

| Condicao | Causa | Correcao |
|----------|-------|----------|
| Todos os upstreams nao saudaveis | Problema de conectividade de rede | Verifique se o no consegue alcancar os IPs upstream |
| Alta latencia + SERVFAIL | Timeout do upstream | Aumente o timeout na configuracao de health check |
| SERVFAIL intermitente | Engine DNS sobrecarregado | Aumente os limites de recursos do agent |

### ConfigMap esta vazio apos criar um pool

**Diagnostico:**

```bash
# Verificar logs do operator
kubectl logs -n astradns-system -l app.kubernetes.io/component=operator --tail=100

# Verificar status do pool
kubectl get dnsupstreampool <name> -n astradns-system -o yaml
```

**Causas comuns:**

- Pool tem spec invalida (verifique as condicoes de status para `InvalidSpec`)
- Operator nao esta em execucao
- Problema de RBAC (operator nao consegue escrever ConfigMaps)

### DNS funciona mas metricas do AstraDNS estao em zero

**Causa:** O CoreDNS nao esta encaminhando para o AstraDNS.

**Correcao:** Consulte [Integracao com CoreDNS](../guides/coredns-integration.md) para verificar a configuracao de encaminhamento.

## Coletando Informacoes para Depuracao

Para relatorios de bugs, colete estas informacoes:

```bash
# Informacoes de versao
helm list -n astradns-system
kubectl get pods -n astradns-system -o wide

# Logs do operator
kubectl logs -n astradns-system -l app.kubernetes.io/component=operator --tail=200

# Logs do agent
kubectl logs -n astradns-system -l app.kubernetes.io/component=agent --tail=200

# Status dos CRDs
kubectl get dnsupstreampools,dnscacheprofiles,externaldnspolicies \
  -n astradns-system -o yaml

# ConfigMap
kubectl get configmap astradns-agent-config -n astradns-system -o yaml

# Configuracao do CoreDNS
kubectl get configmap coredns -n kube-system -o yaml

# Informacoes dos nos
kubectl get nodes -o wide
```
