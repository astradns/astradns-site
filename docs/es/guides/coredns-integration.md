# Integración con CoreDNS

Esta guía explica cómo AstraDNS aplica un patch en CoreDNS para reenviar tráfico DNS externo a través del Agent, con comportamiento específico para `node-local` y `central`.

---

## Cómo funciona

Cuando `clusterDNS.forwardExternalToAstraDNS.enabled=true`, el chart crea un Job post-install/post-upgrade que:

1. lee el `Corefile` actual del ConfigMap de CoreDNS;
2. guarda un backup en `Corefile.astradns.backup`;
3. reescribe la directiva `forward` para apuntar a AstraDNS;
4. opcionalmente reinicia el Deployment de CoreDNS.

---

## Configuración según perfil

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

### central con ClusterIP fija

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

### central con auto-descubrimiento de Service IP

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

En este modo, el Job de patch consulta el Service DNS del Agent en runtime e inyecta el `clusterIP` descubierto en CoreDNS.

---

## Bloque `forward` resultante

Con fallback habilitado, CoreDNS usa un patrón similar a:

```text
forward . <astradns-target> /etc/resolv.conf {
    policy sequential
}
```

`policy sequential` mantiene AstraDNS como primario y cae automáticamente al resolver del nodo si AstraDNS deja de estar disponible.

---

## Guardrails

- `node-local` + patch de CoreDNS requiere `agent.network.mode=linkLocal`.
- `central` rechaza `agent.network.mode=linkLocal`.
- `clusterDNS.provider` actualmente soporta solo `coredns`.
- `kubectlImagePullPolicy` debe ser `Always`, `IfNotPresent` o `Never`.

---

## Validación post-despliegue

```bash
# 1) Corefile final
kubectl -n kube-system get configmap coredns -o jsonpath='{.data.Corefile}'

# 2) Presencia del backup
kubectl -n kube-system get configmap coredns -o jsonpath='{.data.Corefile\.astradns\.backup}'

# 3) Ejecución del Job de patch
kubectl -n astradns-system get jobs | grep coredns-patch
kubectl -n astradns-system logs job/<release>-astradns-coredns-patch

# 4) Prueba de humo DNS
kubectl run dns-test --rm -it --restart=Never --image=busybox:1.37 -- nslookup example.com
```

En `central`, valida también el endpoint del Service:

```bash
kubectl -n astradns-system get svc <release>-astradns-agent-dns -o wide
```

---

## Rollback manual

```bash
kubectl -n kube-system patch configmap coredns --type merge --patch-file <(cat <<'EOF'
data:
  Corefile: |
    # pega aquí el contenido de Corefile.astradns.backup
EOF
)

kubectl -n kube-system rollout restart deployment coredns
```

---

## Relacionado

- [Perfiles de Topología](topology-profiles.md)
- [Runbook](../operations/runbook.md)
