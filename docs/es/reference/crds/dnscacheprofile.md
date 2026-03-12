# DNSCacheProfile

Configura el comportamiento del caché DNS incluyendo límites de tamaño, límites de TTL y umbrales de prefetch.

## Ejemplo

```yaml
apiVersion: dns.astradns.com/v1alpha1
kind: DNSCacheProfile
metadata:
  name: default
  namespace: astradns-system
spec:
  maxEntries: 50000
  positiveTtl:
    minSeconds: 60
    maxSeconds: 300
  negativeTtl:
    seconds: 30
  prefetch:
    enabled: true
    threshold: 5
```

!!! note "Descubrimiento automático"
    El operator usa automáticamente el perfil de caché llamado `default` en el mismo namespace que el pool de upstreams. No se necesita una referencia explícita.

## Spec

### `maxEntries` (opcional)

| Campo | Tipo | Por Defecto | Validación | Descripción |
|-------|------|-------------|------------|-------------|
| `maxEntries` | int32 | `100000` | min 1 | Número máximo de entradas en el caché DNS |

### `positiveTtl` (opcional)

Controla el ajuste de TTL para respuestas exitosas (NOERROR).

| Campo | Tipo | Por Defecto | Validación | Descripción |
|-------|------|-------------|------------|-------------|
| `minSeconds` | int32 | `60` | min 1 | TTL mínimo aplicado a las respuestas en caché |
| `maxSeconds` | int32 | `300` | min 1, debe ser >= minSeconds | TTL máximo aplicado a las respuestas en caché |

!!! info "Validación entre campos"
    El CRD impone `maxSeconds >= minSeconds` mediante validación CEL. Crear un perfil con `maxSeconds < minSeconds` se rechaza en tiempo de admisión.

### `negativeTtl` (opcional)

| Campo | Tipo | Por Defecto | Validación | Descripción |
|-------|------|-------------|------------|-------------|
| `seconds` | int32 | `30` | min 1 | TTL para respuestas negativas (NXDOMAIN) |

### `prefetch` (opcional)

El prefetch re-resuelve dominios populares antes de que expire su TTL, manteniéndolos en caché.

| Campo | Tipo | Por Defecto | Validación | Descripción |
|-------|------|-------------|------------|-------------|
| `enabled` | bool | `true` | -- | Habilitar prefetch para dominios populares |
| `threshold` | int32 | `10` | min 1 | Número de consultas antes de que un dominio sea considerado popular |

## Status

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `conditions` | []Condition | Condiciones estándar |

### Conditions

| Tipo | Estado | Razón | Significado |
|------|--------|-------|-------------|
| `Active` | `True` | `Valid` | Perfil validado exitosamente |
