# Referencia de CRDs

O AstraDNS define tres Custom Resource Definitions no grupo de API `dns.astradns.com/v1alpha1`.

| CRD | Descricao Resumida | Escopo |
|-----|---------------------|--------|
| [`DNSUpstreamPool`](dnsupstreampool.md) | Resolvers upstream, health checks, balanceamento de carga | Namespaced |
| [`DNSCacheProfile`](dnscacheprofile.md) | Tamanho do cache, limites de TTL, prefetch | Namespaced |
| [`ExternalDNSPolicy`](externaldnspolicy.md) | Mapeamento de namespace para pool | Namespaced |

## Grupo de API

```
dns.astradns.com/v1alpha1
```

!!! warning "API Alpha"
    A API `v1alpha1` esta em desenvolvimento ativo. Nomes de campos e semantica podem mudar em versoes futuras. Consulte o [guia de atualizacao](../../operations/upgrade-guide.md) para instrucoes de migracao.

## Condicoes de Status

Todos os tres CRDs utilizam o padrao de Conditions do Kubernetes:

```yaml
status:
  conditions:
    - type: Ready          # ou Active, Validated
      status: "True"       # ou "False"
      reason: Ready        # razao legivel por maquina
      message: "..."       # mensagem legivel por humanos
      lastTransitionTime: "2026-03-11T10:00:00Z"
      observedGeneration: 1
```
