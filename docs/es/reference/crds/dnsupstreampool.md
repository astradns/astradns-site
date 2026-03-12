# DNSUpstreamPool

Define un pool de resolvers DNS upstream con configuración de verificación de salud y balanceo de carga.

## Ejemplo

```yaml
apiVersion: dns.astradns.com/v1alpha1
kind: DNSUpstreamPool
metadata:
  name: production
  namespace: astradns-system
spec:
  upstreams:
    - address: "1.1.1.1"
      port: 53
    - address: "8.8.8.8"
      port: 53
    - address: "9.9.9.9"
  healthCheck:
    enabled: true
    intervalSeconds: 30
    timeoutSeconds: 5
    failureThreshold: 3
  loadBalancing:
    strategy: round-robin
```

## Spec

### `upstreams` (requerido)

Lista de resolvers DNS upstream. Se requiere al menos un upstream.

| Campo | Tipo | Por Defecto | Validación | Descripción |
|-------|------|-------------|------------|-------------|
| `address` | string | -- | Requerido, longitud mínima 1 | Dirección IP o nombre de host del resolver upstream |
| `port` | int32 | `53` | 1-65535 | Número de puerto |

### `healthCheck` (opcional)

| Campo | Tipo | Por Defecto | Validación | Descripción |
|-------|------|-------------|------------|-------------|
| `enabled` | bool | `true` | -- | Habilitar sondeo periódico de salud |
| `intervalSeconds` | int32 | `30` | min 1 | Segundos entre verificaciones de salud |
| `timeoutSeconds` | int32 | `5` | min 1 | Tiempo de espera para cada sondeo |
| `failureThreshold` | int32 | `3` | min 1 | Fallas consecutivas antes de marcar como no saludable |

### `loadBalancing` (opcional)

| Campo | Tipo | Por Defecto | Validación | Descripción |
|-------|------|-------------|------------|-------------|
| `strategy` | string | `round-robin` | Enum: `round-robin`, `first-available`, `random` | Estrategia de balanceo de carga entre upstreams |

## Status

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `conditions` | []Condition | Condiciones estándar (ver abajo) |
| `upstreamStatuses` | []UpstreamStatus | Salud y latencia por upstream |

### Conditions

| Tipo | Estado | Razón | Significado |
|------|--------|-------|-------------|
| `Ready` | `True` | `Ready` | Configuración renderizada exitosamente |
| `Ready` | `False` | `Superseded` | Otro pool está activo en este namespace |
| `Ready` | `False` | `InvalidSpec` | La validación de la especificación falló |
| `Ready` | `False` | `ConfigGenerationFailed` | Error en la generación de configuración del motor |
| `Ready` | `False` | `ValidationFailed` | La configuración renderizada falló en la validación |
| `Ready` | `False` | `ProfileLookupFailed` | No se pudo obtener el perfil de caché por defecto |

### UpstreamStatus

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `address` | string | Dirección del upstream |
| `healthy` | bool | Estado de salud actual |
| `latencyMs` | int64 | Latencia del último sondeo en milisegundos |

## Comportamiento Multi-Pool

Solo **un pool por namespace** está activo a la vez. Cuando existen múltiples pools, el pool activo se selecciona por:

1. **`creationTimestamp` más antiguo** -- el pool creado primero gana
2. **ResourceVersion inicial más bajo** -- desempate para pools creados en el mismo segundo
3. **Nombre alfabético** -- desempate determinístico final

Los pools no activos reciben la condición `Superseded` con un mensaje indicando qué pool está activo.

!!! tip "Webhook de validación"
    Habilite el webhook de validación (`webhook.enabled=true`) para rechazar la creación de un segundo pool en el mismo namespace en tiempo de admisión.
