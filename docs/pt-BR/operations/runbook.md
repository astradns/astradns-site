# Runbook

Procedimentos operacionais para diagnosticar e resolver problemas do AstraDNS.

## Verificacoes Primarias de Saude

Execute estas verificacoes primeiro ao investigar qualquer problema de DNS:

```bash
# 1. Todos os pods estao em execucao?
kubectl get pods -n astradns-system

# 2. Os CRDs estao saudaveis?
kubectl get dnsupstreampools -n astradns-system
kubectl get dnscacheprofiles -n astradns-system

# 3. O ConfigMap esta preenchido?
kubectl get configmap astradns-agent-config -n astradns-system \
  -o jsonpath='{.data.config\.json}' | jq .

# 4. As metricas estao fluindo?
kubectl port-forward -n astradns-system ds/astradns-agent 9153:9153 &
curl -s http://localhost:9153/metrics | grep astradns_queries_total

# 5. O ServiceMonitor esta coletando?
kubectl get servicemonitor -n astradns-system
```

## Incidentes Comuns

### Consultas DNS falham apos implantacao

**Sintomas:** Pods recebem SERVFAIL ou timeout ao resolver dominios externos.

**Diagnostico:**

```bash
# Verificar saude do agent
kubectl logs -n astradns-system -l app.kubernetes.io/component=agent --tail=50

# Verificar se o engine esta em execucao
kubectl exec -n astradns-system ds/astradns-agent -- ps aux | grep -E 'unbound|coredns|pdns'

# Verificar readiness
kubectl get pods -n astradns-system -l app.kubernetes.io/component=agent \
  -o jsonpath='{range .items[*]}{.metadata.name}: {.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'

# Teste DNS direto para o agent
kubectl run dns-debug --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup -port=5353 example.com <agent-pod-ip>
```

**Resolucao:**

1. Se o processo do engine nao esta em execucao: verifique os logs do agent para erros de inicializacao, valide que o ConfigMap tem JSON valido
2. Se o engine retorna SERVFAIL: verifique se os resolvers upstream sao acessiveis a partir do no
3. Se o pod do agent esta em CrashLoop: verifique os limites de recursos, confirme que o binario do engine existe na imagem

### CoreDNS nao foi modificado

**Sintomas:** DNS funciona, mas as consultas ignoram o AstraDNS (metricas nao incrementam).

**Diagnostico:**

```bash
# Verificar se o Corefile do CoreDNS tem a diretiva forward
kubectl get configmap coredns -n kube-system -o yaml | grep -A2 forward

# Verificar o status do job de patch
kubectl get jobs -n kube-system | grep astradns
kubectl logs -n kube-system job/astradns-coredns-patch
```

**Resolucao:**

1. Re-execute o job de patch: `kubectl delete job astradns-coredns-patch -n kube-system` e depois `helm upgrade ...`
2. Patch manual: edite o ConfigMap do CoreDNS para adicionar `forward . 169.254.20.11`
3. Reinicie o CoreDNS: `kubectl rollout restart deployment coredns -n kube-system`

### Webhook bloqueia criacao legitima de pool

**Sintomas:** `kubectl apply` retorna erro de admissao sobre pool existente.

**Diagnostico:**

```bash
# Listar pools existentes no namespace
kubectl get dnsupstreampools -n <namespace>

# Verificar configuracao do webhook
kubectl get validatingwebhookconfigurations | grep astradns
```

**Resolucao:**

O webhook impoe um unico `DNSUpstreamPool` por namespace. Para alterar a configuracao:

1. **Atualize** o pool existente em vez de criar um novo
2. **Exclua** o pool antigo primeiro e depois crie o novo
3. Se o webhook estiver bloqueando todas as operacoes, desabilite temporariamente: `helm upgrade ... --set webhook.enabled=false`

### Reload de configuracao falha

**Sintomas:** `astradns_agent_config_reload_errors_total` esta aumentando.

**Diagnostico:**

```bash
# Verificar logs do agent para erros de reload
kubectl logs -n astradns-system -l app.kubernetes.io/component=agent | grep -i reload

# Validar conteudo do ConfigMap
kubectl get configmap astradns-agent-config -n astradns-system \
  -o jsonpath='{.data.config\.json}' | python3 -m json.tool
```

**Resolucao:**

1. Corrija a spec do CRD do upstream pool (verifique enderecos/portas invalidos)
2. O operator vai re-renderizar e atualizar o ConfigMap
3. O agent detectara a alteracao e fara reload automaticamente

## Procedimento de Rollback

### Rollback via Helm

```bash
# Listar revisoes
helm history astradns -n astradns-system

# Rollback para a revisao anterior
helm rollback astradns <revision> -n astradns-system
```

### Verificacao Pos-Rollback

```bash
# Verificar se os pods estao em execucao
kubectl get pods -n astradns-system

# Verificar resolucao DNS
kubectl run dns-test --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup example.com

# Verificar se as metricas estao fluindo
curl -s http://localhost:9153/metrics | grep astradns_queries_total
```

### Emergencia: Desabilitar AstraDNS Sem Desinstalar

Se o AstraDNS estiver causando problemas de DNS em todo o cluster e voce precisar restaurar o DNS imediatamente:

```bash
# Reverter o CoreDNS para a configuracao original
kubectl get configmap coredns-backup-astradns -n kube-system \
  -o jsonpath='{.data.Corefile}' > /tmp/original-corefile

kubectl create configmap coredns -n kube-system \
  --from-file=Corefile=/tmp/original-corefile \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl rollout restart deployment coredns -n kube-system
```

Isso restaura o CoreDNS para sua configuracao anterior ao AstraDNS. Os pods do AstraDNS continuarao em execucao, mas nao receberao mais consultas.
