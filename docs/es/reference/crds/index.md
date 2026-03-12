# Referencia de CRDs

AstraDNS define tres Custom Resource Definitions en el grupo de API `dns.astradns.com/v1alpha1`.

| CRD | Descripción Breve | Alcance |
|-----|-------------------|---------|
| [`DNSUpstreamPool`](dnsupstreampool.md) | Resolvers upstream, verificaciones de salud, balanceo de carga | Namespaced |
| [`DNSCacheProfile`](dnscacheprofile.md) | Tamaño de caché, límites de TTL, prefetch | Namespaced |
| [`ExternalDNSPolicy`](externaldnspolicy.md) | Mapeo de namespace a pool | Namespaced |

## Grupo de API

```
dns.astradns.com/v1alpha1
```

!!! warning "API Alpha"
    La API `v1alpha1` está en desarrollo activo. Los nombres de campos y la semántica pueden cambiar en versiones futuras. Consulte la [guía de actualización](../../operations/upgrade-guide.md) para instrucciones de migración.

## Condiciones de Estado

Los tres CRDs usan el patrón estándar de Conditions de Kubernetes:

```yaml
status:
  conditions:
    - type: Ready          # or Active, Validated
      status: "True"       # or "False"
      reason: Ready        # machine-readable reason
      message: "..."       # human-readable message
      lastTransitionTime: "2026-03-11T10:00:00Z"
      observedGeneration: 1
```
