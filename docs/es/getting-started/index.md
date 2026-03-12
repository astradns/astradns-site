# Primeros Pasos

AstraDNS se instala como un chart de Helm y no requiere cambios en el código de su aplicación. Esta guía lo guiará a través de una configuración mínima en cualquier clúster de Kubernetes.

## Qué Obtiene

Después de la instalación, cada nodo de su clúster ejecuta un AstraDNS Agent que:

- **Enruta** las consultas DNS externas a través de un resolver administrado (Unbound por defecto)
- **Almacena en caché** las respuestas con TTLs configurables
- **Registra** cada consulta como JSON estructurado
- **Exporta** métricas de Prometheus sobre aciertos de caché, latencia, fallas y salud de los upstreams
- **Verifica la salud** de los resolvers upstream y reporta el estado mediante condiciones del CRD

## Instalación en 5 Minutos

### 1. Agregar el chart de Helm

```bash
helm install astradns deploy/helm/astradns \
  --namespace astradns-system \
  --create-namespace
```

### 2. Crear un pool de upstreams

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

### 3. Verificar

```bash
# Check operator is running
kubectl get pods -n astradns-system -l app.kubernetes.io/component=operator

# Check agent is running on nodes
kubectl get pods -n astradns-system -l app.kubernetes.io/component=agent

# Check the pool status
kubectl get dnsupstreampools -n astradns-system
```

El pool debería mostrar `Ready=True` en segundos.

### 4. Probar la resolución DNS

```bash
kubectl run dns-test --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup example.com
```

## Próximos Pasos

- [Prerrequisitos](prerequisites.md) para clústeres de producción
- [Instalación](installation.md) con valores de producción
- [Configuración de Helm](../guides/helm-configuration.md) referencia
- [Integración con CoreDNS](../guides/coredns-integration.md) para DNS local en el nodo
