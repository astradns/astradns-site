# Prerrequisitos

## Requisitos del Clúster

| Requisito | Mínimo | Recomendado |
|-----------|--------|-------------|
| Kubernetes | v1.30+ | v1.32+ |
| Helm | v3.12+ | v3.16+ |
| Runtime de contenedores | Cualquiera compatible con OCI | containerd |
| CNI | Cualquiera | Calico, Cilium o Flannel |

## Dependencias Opcionales

| Componente | Requerido Para | Instalación |
|------------|---------------|-------------|
| cert-manager | TLS del webhook de validación | [cert-manager.io](https://cert-manager.io/docs/installation/) |
| Prometheus Operator | Scraping mediante ServiceMonitor | [prometheus-operator](https://github.com/prometheus-operator/prometheus-operator) |
| Grafana | Dashboards de DNS | Cualquier Grafana con provisionamiento por sidecar |

## Requisitos de Recursos

### Operator (Deployment)

El operator es liviano: observa los CRDs y escribe ConfigMaps.

| Recurso | Request | Limit |
|---------|---------|-------|
| CPU | 100m | 200m |
| Memoria | 128Mi | 256Mi |

### Agent (DaemonSet)

El agent se ejecuta en cada nodo y maneja el tráfico DNS.

| Recurso | Request | Limit |
|---------|---------|-------|
| CPU | 100m | 500m |
| Memoria | 128Mi | 512Mi |

!!! tip "Dimensionamiento para nodos de alto tráfico"
    Los nodos que sirven más de 5,000 QPS pueden necesitar límites de CPU más altos. Monitoree la tasa de `astradns_queries_total` para identificar nodos con alta carga.

## Requisitos de Red

### Modo hostPort (por defecto)

El agent se enlaza al puerto `5353` en cada nodo mediante `hostPort`. No se necesita configuración de red especial.

### Modo linkLocal (recomendado para producción)

El agent se enlaza a la dirección link-local `169.254.20.11:5353` en cada nodo. Este modo requiere:

- `hostNetwork: true` en el DaemonSet del agent (configurado automáticamente por Helm)
- CoreDNS configurado para reenviar consultas externas a `169.254.20.11:5353` (automatizado mediante `clusterDNS.forwardExternalToAstraDNS.enabled=true`)

## RBAC

AstraDNS crea los siguientes recursos RBAC con alcance de clúster:

- **ClusterRole** para el operator (leer CRDs, escribir ConfigMaps, crear eventos)
- **ClusterRoleBinding** vinculando el ServiceAccount del operator
- **ServiceAccount** por componente (operator, agent)

No se requieren privilegios de cluster-admin.
