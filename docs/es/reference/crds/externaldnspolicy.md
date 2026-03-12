# ExternalDNSPolicy

Mapea namespaces a pools de upstreams y perfiles de caché, habilitando políticas de resolución DNS por namespace.

## Ejemplo

```yaml
apiVersion: dns.astradns.com/v1alpha1
kind: ExternalDNSPolicy
metadata:
  name: production-policy
  namespace: astradns-system
spec:
  selector:
    namespaces:
      - production
      - staging
  upstreamPoolRef:
    name: production
  cacheProfileRef:
    name: default
```

!!! warning "Alcance de aplicación"
    En la versión actual, `ExternalDNSPolicy` valida referencias y establece condiciones de estado, pero no impone enrutamiento a nivel de namespace. La imposición completa requiere la ruta de datos directa (planificada para una versión futura).

## Spec

### `selector` (requerido)

| Campo | Tipo | Por Defecto | Validación | Descripción |
|-------|------|-------------|------------|-------------|
| `namespaces` | []string | -- | Requerido, mínimo 1 elemento | Lista de nombres de namespaces a los que se aplica esta política |

### `upstreamPoolRef` (requerido)

| Campo | Tipo | Por Defecto | Validación | Descripción |
|-------|------|-------------|------------|-------------|
| `name` | string | -- | Requerido, longitud mínima 1 | Nombre del `DNSUpstreamPool` a utilizar |

### `cacheProfileRef` (opcional)

| Campo | Tipo | Por Defecto | Validación | Descripción |
|-------|------|-------------|------------|-------------|
| `name` | string | -- | Longitud mínima 1 | Nombre del `DNSCacheProfile` a utilizar |

## Status

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `conditions` | []Condition | Condiciones estándar |
| `appliedNodes` | int32 | Número de nodos donde esta política está activa |

### Conditions

| Tipo | Estado | Razón | Significado |
|------|--------|-------|-------------|
| `Validated` | `True` | `Valid` | Todas las referencias se resolvieron exitosamente |
| `Validated` | `False` | `UpstreamPoolNotFound` | El pool referenciado no existe |
| `Validated` | `False` | `CacheProfileNotFound` | El perfil de caché referenciado no existe |
