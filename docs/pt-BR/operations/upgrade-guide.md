# Guia de Atualizacao

## Processo de Atualizacao

### 1. Revise as Notas de Release

Antes de atualizar, revise as notas de release para alteracoes incompativeis, novos recursos e deprecacoes.

### 2. Backup

```bash
# Backup dos CRDs
kubectl get dnsupstreampools,dnscacheprofiles,externaldnspolicies \
  -n astradns-system -o yaml > astradns-backup.yaml

# Backup dos valores Helm
helm get values astradns -n astradns-system > current-values.yaml

# Anote a revisao atual
helm history astradns -n astradns-system
```

### 3. Dry Run

```bash
helm upgrade astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system \
  -f current-values.yaml \
  --dry-run --debug
```

Revise a saida para alteracoes inesperadas.

### 4. Atualizar

```bash
helm upgrade astradns oci://ghcr.io/astradns/helm-charts/astradns \
  --namespace astradns-system \
  -f current-values.yaml
```

### 5. Verificar

```bash
# Verificar se os pods estao em execucao
kubectl get pods -n astradns-system

# Verificar status dos CRDs
kubectl get dnsupstreampools -n astradns-system

# Testar resolucao DNS
kubectl run dns-test --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup example.com

# Verificar metricas
AGENT_POD="$(kubectl -n astradns-system get pods -l app.kubernetes.io/component=agent -o jsonpath='{.items[0].metadata.name}')"
kubectl port-forward -n astradns-system "pod/${AGENT_POD}" 9153:9153 &
curl -s http://localhost:9153/metrics | grep astradns_queries_total
```

## Rollback

Se a atualizacao causar problemas:

```bash
helm rollback astradns <previous-revision> -n astradns-system
```

## Atualizacoes de CRDs

!!! warning "Alteracoes em CRDs requerem atencao manual"
    Se a nova versao incluir alteracoes de CRD, execute o upgrade com renderizacao de CRDs habilitada:

    ```bash
    helm upgrade astradns oci://ghcr.io/astradns/helm-charts/astradns \
      --namespace astradns-system \
      -f current-values.yaml \
      --set crds.install=true
    ```

## Notas Especificas por Versao

### v0.1.x

Release inicial. Sem alteracoes incompativeis dentro da serie v0.1.x.

**Limitacoes conhecidas:**

- Metricas por namespace aparecem como "unknown" (requer caminho de dados direto em release futura)
- ExternalDNSPolicy valida referencias, mas nao aplica roteamento
- Apenas um DNSUpstreamPool por namespace (por design; webhook impoe isso quando habilitado)
