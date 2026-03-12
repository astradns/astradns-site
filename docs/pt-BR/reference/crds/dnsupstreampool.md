# DNSUpstreamPool

Define um pool de resolvers DNS upstream com configuracao de health check e balanceamento de carga.

## Exemplo

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

### `upstreams` (obrigatorio)

Lista de resolvers DNS upstream. Pelo menos um upstream e obrigatorio.

| Campo | Tipo | Padrao | Validacao | Descricao |
|-------|------|--------|-----------|-----------|
| `address` | string | -- | Obrigatorio, comprimento minimo 1 | Endereco IP ou hostname do resolver upstream |
| `port` | int32 | `53` | 1-65535 | Numero da porta |

### `healthCheck` (opcional)

| Campo | Tipo | Padrao | Validacao | Descricao |
|-------|------|--------|-----------|-----------|
| `enabled` | bool | `true` | -- | Habilitar verificacao periodica de saude |
| `intervalSeconds` | int32 | `30` | min 1 | Segundos entre verificacoes de saude |
| `timeoutSeconds` | int32 | `5` | min 1 | Timeout para cada sondagem |
| `failureThreshold` | int32 | `3` | min 1 | Falhas consecutivas antes de marcar como nao saudavel |

### `loadBalancing` (opcional)

| Campo | Tipo | Padrao | Validacao | Descricao |
|-------|------|--------|-----------|-----------|
| `strategy` | string | `round-robin` | Enum: `round-robin`, `first-available`, `random` | Estrategia de balanceamento de carga entre upstreams |

## Status

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `conditions` | []Condition | Condicoes padrao (veja abaixo) |
| `upstreamStatuses` | []UpstreamStatus | Saude e latencia por upstream |

### Condicoes

| Tipo | Status | Razao | Significado |
|------|--------|-------|-------------|
| `Ready` | `True` | `Ready` | Configuracao renderizada com sucesso |
| `Ready` | `False` | `Superseded` | Outro pool esta ativo neste namespace |
| `Ready` | `False` | `InvalidSpec` | Validacao da spec falhou |
| `Ready` | `False` | `ConfigGenerationFailed` | Erro na geracao da config do engine |
| `Ready` | `False` | `ValidationFailed` | Configuracao renderizada falhou na validacao |
| `Ready` | `False` | `ProfileLookupFailed` | Nao foi possivel buscar o perfil de cache padrao |

### UpstreamStatus

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `address` | string | Endereco do upstream |
| `healthy` | bool | Status atual de saude |
| `latencyMs` | int64 | Latencia da ultima sondagem em milissegundos |

## Comportamento Multi-Pool

Apenas **um pool por namespace** esta ativo por vez. Quando multiplos pools existem, o pool ativo e selecionado por:

1. **`creationTimestamp` mais antigo** -- o pool criado primeiro vence
2. **Menor ResourceVersion inicial** -- desempate para pools criados no mesmo segundo
3. **Nome alfabetico** -- desempate deterministico final

Pools nao ativos recebem a condicao `Superseded` com uma mensagem indicando qual pool esta ativo.

!!! tip "Webhook de validacao"
    Habilite o webhook de validacao (`webhook.enabled=true`) para rejeitar a criacao de um segundo pool no mesmo namespace no momento da admissao.
