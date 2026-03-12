# DNSCacheProfile

Configura o comportamento do cache DNS incluindo limites de tamanho, limites de TTL e thresholds de prefetch.

## Exemplo

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

!!! note "Descoberta automatica"
    O operator utiliza automaticamente o perfil de cache com o nome `default` no mesmo namespace do upstream pool. Nenhuma referencia explicita e necessaria.

## Spec

### `maxEntries` (opcional)

| Campo | Tipo | Padrao | Validacao | Descricao |
|-------|------|--------|-----------|-----------|
| `maxEntries` | int32 | `100000` | min 1 | Numero maximo de entradas no cache DNS |

### `positiveTtl` (opcional)

Controla o ajuste de TTL para respostas bem-sucedidas (NOERROR).

| Campo | Tipo | Padrao | Validacao | Descricao |
|-------|------|--------|-----------|-----------|
| `minSeconds` | int32 | `60` | min 1 | TTL minimo aplicado as respostas em cache |
| `maxSeconds` | int32 | `300` | min 1, deve ser >= minSeconds | TTL maximo aplicado as respostas em cache |

!!! info "Validacao entre campos"
    O CRD impoe `maxSeconds >= minSeconds` via validacao CEL. Criar um perfil com `maxSeconds < minSeconds` e rejeitado no momento da admissao.

### `negativeTtl` (opcional)

| Campo | Tipo | Padrao | Validacao | Descricao |
|-------|------|--------|-----------|-----------|
| `seconds` | int32 | `30` | min 1 | TTL para respostas negativas (NXDOMAIN) |

### `prefetch` (opcional)

O prefetch re-resolve dominios populares antes que seu TTL expire, mantendo-os em cache.

| Campo | Tipo | Padrao | Validacao | Descricao |
|-------|------|--------|-----------|-----------|
| `enabled` | bool | `true` | -- | Habilitar prefetch para dominios populares |
| `threshold` | int32 | `10` | min 1 | Numero de consultas antes de um dominio ser considerado popular |

## Status

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `conditions` | []Condition | Condicoes padrao |

### Condicoes

| Tipo | Status | Razao | Significado |
|------|--------|-------|-------------|
| `Active` | `True` | `Valid` | Perfil validado com sucesso |
