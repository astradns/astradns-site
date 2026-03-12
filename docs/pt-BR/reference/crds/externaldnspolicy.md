# ExternalDNSPolicy

Mapeia namespaces para pools de upstreams e perfis de cache, habilitando politicas de resolucao DNS por namespace.

## Exemplo

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

!!! warning "Escopo de aplicacao"
    Na versao atual, o `ExternalDNSPolicy` valida referencias e define condicoes de status, mas nao aplica roteamento por namespace. A aplicacao completa requer o caminho de dados direto (planejado para uma versao futura).

## Spec

### `selector` (obrigatorio)

| Campo | Tipo | Padrao | Validacao | Descricao |
|-------|------|--------|-----------|-----------|
| `namespaces` | []string | -- | Obrigatorio, minimo 1 item | Lista de nomes de namespaces aos quais esta politica se aplica |

### `upstreamPoolRef` (obrigatorio)

| Campo | Tipo | Padrao | Validacao | Descricao |
|-------|------|--------|-----------|-----------|
| `name` | string | -- | Obrigatorio, comprimento minimo 1 | Nome do `DNSUpstreamPool` a ser utilizado |

### `cacheProfileRef` (opcional)

| Campo | Tipo | Padrao | Validacao | Descricao |
|-------|------|--------|-----------|-----------|
| `name` | string | -- | comprimento minimo 1 | Nome do `DNSCacheProfile` a ser utilizado |

## Status

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `conditions` | []Condition | Condicoes padrao |
| `appliedNodes` | int32 | Numero de nos onde esta politica esta ativa |

### Condicoes

| Tipo | Status | Razao | Significado |
|------|--------|-------|-------------|
| `Validated` | `True` | `Valid` | Todas as referencias resolvidas com sucesso |
| `Validated` | `False` | `UpstreamPoolNotFound` | Pool referenciado nao existe |
| `Validated` | `False` | `CacheProfileNotFound` | Perfil de cache referenciado nao existe |
