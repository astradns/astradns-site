# Integración con CoreDNS

AstraDNS se integra con el CoreDNS del clúster para interceptar las consultas DNS externas. Esta guía explica cómo funciona la integración y cómo configurarla.

## Cómo Funciona

Cuando `coredns.integration.enabled=true`, el chart de Helm crea un Job post-instalación que:

1. Crea RBAC (ServiceAccount, Role, RoleBinding) en `kube-system`
2. Lee el ConfigMap actual de CoreDNS
3. Respalda el Corefile original en un nuevo ConfigMap
4. Agrega una directiva `forward` que enruta las consultas no internas del clúster hacia AstraDNS
5. Reinicia el deployment de CoreDNS para aplicar el cambio

### Antes de la Integración

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

### Después de la Integración

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

## Configuración

### Habilitar mediante Helm

```yaml
agent:
  network:
    mode: linkLocal                   # Required for CoreDNS integration
    linkLocalIP: 169.254.20.11

coredns:
  integration:
    enabled: true
```

### Modo de Red Requerido

La integración con CoreDNS requiere el modo de red `linkLocal`. El agent se enlaza a `169.254.20.11:53` en cada nodo, proporcionando una dirección consistente para el reenvío de CoreDNS.

!!! warning
    Usar el modo `hostPort` con la integración de CoreDNS no está soportado porque CoreDNS necesitaría conocer la dirección IP de cada nodo, la cual varía en todo el clúster.

## Verificación

Después de la instalación, verifique la integración:

```bash
# Check the CoreDNS ConfigMap was patched
kubectl get configmap coredns -n kube-system -o yaml | grep forward

# Check the backup exists
kubectl get configmap coredns-backup-astradns -n kube-system

# Test DNS resolution through AstraDNS
kubectl run dns-test --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup example.com
```

Verifique las métricas del agent para confirmar que las consultas están fluyendo:

```bash
kubectl port-forward -n astradns-system ds/astradns-agent 9153:9153
curl -s http://localhost:9153/metrics | grep astradns_queries_total
```

## Reversión

Para revertir manualmente la integración con CoreDNS:

```bash
# Restore from backup
kubectl get configmap coredns-backup-astradns -n kube-system \
  -o jsonpath='{.data.Corefile}' | \
  kubectl create configmap coredns -n kube-system \
  --from-file=Corefile=/dev/stdin --dry-run=client -o yaml | \
  kubectl apply -f -

# Restart CoreDNS
kubectl rollout restart deployment coredns -n kube-system
```

O simplemente desinstale AstraDNS. El proceso de `helm uninstall` no revierte automáticamente los cambios en CoreDNS, por lo que puede ser necesaria una reversión manual.
