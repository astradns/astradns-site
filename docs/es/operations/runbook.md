# Runbook

Procedimientos operativos para diagnosticar y resolver problemas de AstraDNS.

## Verificaciones de Salud Primarias

Ejecute estas verificaciones primero al investigar cualquier problema de DNS:

```bash
# 1. Are all pods running?
kubectl get pods -n astradns-system

# 2. Are CRDs healthy?
kubectl get dnsupstreampools -n astradns-system
kubectl get dnscacheprofiles -n astradns-system

# 3. Is the ConfigMap populated?
kubectl get configmap astradns-agent-config -n astradns-system \
  -o jsonpath='{.data.config\.json}' | jq .

# 4. Are metrics flowing?
AGENT_POD="$(kubectl -n astradns-system get pods -l app.kubernetes.io/component=agent -o jsonpath='{.items[0].metadata.name}')"
kubectl port-forward -n astradns-system "pod/${AGENT_POD}" 9153:9153 &
curl -s http://localhost:9153/metrics | grep astradns_queries_total

# 5. Is the ServiceMonitor scraping?
kubectl get servicemonitor -n astradns-system
```

## Incidentes Comunes

### Las consultas DNS fallan después del despliegue

**Síntomas:** Los pods obtienen SERVFAIL o timeout al resolver dominios externos.

**Diagnóstico:**

```bash
# Check agent health
kubectl logs -n astradns-system -l app.kubernetes.io/component=agent --tail=50

# Check if engine is running
AGENT_POD="$(kubectl -n astradns-system get pods -l app.kubernetes.io/component=agent -o jsonpath='{.items[0].metadata.name}')"
kubectl exec -n astradns-system "pod/${AGENT_POD}" -- ps aux | grep -E 'unbound|coredns|pdns|named'

# Check readiness
kubectl get pods -n astradns-system -l app.kubernetes.io/component=agent \
  -o jsonpath='{range .items[*]}{.metadata.name}: {.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'

# Direct DNS test to agent
kubectl run dns-debug --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup -port=5353 example.com <agent-pod-ip>
```

**Resolución:**

1. Si el proceso del motor no está corriendo: verifique los logs del agent en busca de errores de inicio, compruebe que el ConfigMap tenga JSON válido
2. Si el motor devuelve SERVFAIL: verifique que los resolvers upstream sean alcanzables desde el nodo
3. Si el pod del agent está en CrashLoop: revise los límites de recursos, verifique que el binario del motor exista en la imagen

### CoreDNS no fue parcheado

**Síntomas:** El DNS funciona pero las consultas evitan AstraDNS (las métricas no incrementan).

**Diagnóstico:**

```bash
# Check if CoreDNS Corefile has the forward directive
kubectl get configmap coredns -n kube-system -o yaml | grep -A2 forward

# Check the patch job status
kubectl get jobs -n astradns-system | grep coredns-patch
kubectl logs -n astradns-system job/<helm-fullname>-coredns-patch
```

**Resolución:**

1. Re-ejecute el job de parcheo: `kubectl delete job -n astradns-system <helm-fullname>-coredns-patch` y luego ejecute `helm upgrade ... --set clusterDNS.forwardExternalToAstraDNS.enabled=true`
2. Parcheo manual: edite el ConfigMap de CoreDNS para agregar `forward . <forward-target-configurado>` (en el default linkLocal, `169.254.20.11:5353`)
3. Reinicie CoreDNS: `kubectl rollout restart deployment coredns -n kube-system`

### El webhook bloquea la creación legítima de pools

**Síntomas:** `kubectl apply` devuelve un error de admisión sobre un pool existente.

**Diagnóstico:**

```bash
# List existing pools in the namespace
kubectl get dnsupstreampools -n <namespace>

# Check webhook configuration
kubectl get validatingwebhookconfigurations | grep astradns
```

**Resolución:**

El webhook impone un solo `DNSUpstreamPool` por namespace. Para cambiar la configuración:

1. **Actualice** el pool existente en lugar de crear uno nuevo
2. **Elimine** el pool anterior primero, luego cree el nuevo
3. Si el webhook está bloqueando todas las operaciones, deshabilítelo temporalmente: `helm upgrade ... --set webhook.enabled=false`

### La recarga de configuración falla

**Síntomas:** `astradns_agent_config_reload_errors_total` está incrementando.

**Diagnóstico:**

```bash
# Check agent logs for reload errors
kubectl logs -n astradns-system -l app.kubernetes.io/component=agent | grep -i reload

# Validate ConfigMap content
kubectl get configmap astradns-agent-config -n astradns-system \
  -o jsonpath='{.data.config\.json}' | python3 -m json.tool
```

**Resolución:**

1. Corrija la especificación del CRD del pool de upstreams (verifique direcciones/puertos inválidos)
2. El operator re-renderizará y actualizará el ConfigMap
3. El agent detectará el cambio y se recargará automáticamente

## Procedimiento de Reversión

### Reversión con Helm

```bash
# List revisions
helm history astradns -n astradns-system

# Rollback to previous revision
helm rollback astradns <revision> -n astradns-system
```

### Verificación Post-Reversión

```bash
# Verify pods are running
kubectl get pods -n astradns-system

# Verify DNS resolution
kubectl run dns-test --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup example.com

# Verify metrics are flowing
curl -s http://localhost:9153/metrics | grep astradns_queries_total
```

### Emergencia: Deshabilitar AstraDNS Sin Desinstalar

Si AstraDNS está causando problemas de DNS en todo el clúster y necesita restaurar el DNS de inmediato:

```bash
# Revert CoreDNS to original config
kubectl get configmap coredns -n kube-system \
  -o jsonpath='{.data.Corefile\.astradns\.backup}' > /tmp/original-corefile

test -s /tmp/original-corefile

kubectl create configmap coredns -n kube-system \
  --from-file=Corefile=/tmp/original-corefile \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl rollout restart deployment coredns -n kube-system
```

Esto restaura CoreDNS a su configuración previa a AstraDNS. Los pods de AstraDNS continuarán corriendo pero ya no recibirán consultas.
