# Integração com CoreDNS

Este guia explica como o AstraDNS altera o CoreDNS para encaminhar consultas externas pelo Agent, com comportamento diferente para os perfis `node-local` e `central`.

---

## Como funciona

Quando `clusterDNS.forwardExternalToAstraDNS.enabled=true`, o chart cria um Job pós-install/pós-upgrade que:

1. lê o `Corefile` atual do ConfigMap do CoreDNS;
2. cria backup em `Corefile.astradns.backup`;
3. substitui a diretiva `forward` para apontar ao AstraDNS;
4. opcionalmente reinicia o deployment do CoreDNS.

---

## Configuração por perfil

### node-local

```yaml
agent:
  topology:
    profile: node-local
  network:
    mode: linkLocal
    linkLocalIP: 169.254.20.11

clusterDNS:
  forwardExternalToAstraDNS:
    enabled: true
    namespace: kube-system
    configMapName: coredns
    rolloutDeployment: coredns
    forwardTarget: 169.254.20.11:5353
    fallbackUpstream: /etc/resolv.conf
```

!!! note "Use o target link-local configurado"
    `169.254.20.11` e o padrao do chart.
    Em `node-local`, mantenha `clusterDNS.forwardExternalToAstraDNS.forwardTarget` alinhado com `agent.network.linkLocalIP` (`<linkLocalIP>:5353`).

### central com ClusterIP fixo

```yaml
agent:
  topology:
    profile: central
  deployment:
    replicas: 3
  dnsService:
    type: ClusterIP
    clusterIP: 10.96.0.53
    port: 53

clusterDNS:
  forwardExternalToAstraDNS:
    enabled: true
    namespace: kube-system
    configMapName: coredns
    rolloutDeployment: coredns
    fallbackUpstream: /etc/resolv.conf
```

### central com descoberta automática de Service IP

```yaml
agent:
  topology:
    profile: central
  dnsService:
    type: ClusterIP
    clusterIP: ""
    port: 53

clusterDNS:
  forwardExternalToAstraDNS:
    enabled: true
    namespace: kube-system
    configMapName: coredns
```

Nesse modo, o patch job consulta o Service do Agent e injeta o `clusterIP` real no Corefile.

---

## Exemplo de bloco `forward`

Com fallback habilitado, o CoreDNS fica com padrão semelhante a:

```text
forward . <target-astradns> /etc/resolv.conf {
    policy sequential
}
```

`policy sequential` garante tentativa prioritária no AstraDNS e fallback automático para o resolver original do nó se o Agent estiver indisponível.

---

## Guardrails

- `node-local` + patch CoreDNS exige `agent.network.mode=linkLocal`.
- `central` não aceita `agent.network.mode=linkLocal`.
- `clusterDNS.provider` hoje suporta apenas `coredns`.
- `kubectlImagePullPolicy` deve ser `Always`, `IfNotPresent` ou `Never`.

---

## Validação pós-deploy

```bash
# 1) Corefile final
kubectl -n kube-system get configmap coredns -o jsonpath='{.data.Corefile}'

# 2) Backup gerado
kubectl -n kube-system get configmap coredns -o jsonpath='{.data.Corefile\.astradns\.backup}'

# 3) Job de patch executou
kubectl -n astradns-system get jobs | grep coredns-patch
kubectl -n astradns-system logs job/<helm-fullname>-coredns-patch

# 4) Smoke test DNS
kubectl run dns-test --rm -it --restart=Never --image=busybox:1.37 -- nslookup example.com
```

Em `central`, valide também o Service:

```bash
kubectl -n astradns-system get svc <helm-fullname>-agent-dns -o wide
```

---

## Rollback manual

```bash
kubectl -n kube-system patch configmap coredns --type merge --patch-file <(cat <<'EOF'
data:
  Corefile: |
    # cole aqui o conteúdo salvo em Corefile.astradns.backup
EOF
)

kubectl -n kube-system rollout restart deployment coredns
```

---

## Referências

- [Perfis de Topologia](topology-profiles.md)
- [Runbook](../operations/runbook.md)
