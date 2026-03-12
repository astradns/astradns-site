# Selecao de Engine

O AstraDNS suporta tres engines DNS. O engine e selecionado no momento da implantacao via a variavel de ambiente `ASTRADNS_ENGINE_TYPE` (ou o valor Helm `agent.engineType`).

## Comparacao

| Recurso | Unbound | CoreDNS | PowerDNS Recursor |
|---------|---------|---------|-------------------|
| **Tipo** | Resolver recursivo | Autoritativo + encaminhamento | Resolver recursivo |
| **Linguagem** | C | Go | C++ |
| **Consumo de memoria** | ~20-50 MB | ~30-60 MB | ~40-80 MB |
| **Desempenho de cache** | Excelente | Bom | Bom |
| **Validacao DNSSEC** | Integrada | Baseada em plugin | Integrada |
| **Reload de config** | `unbound-control reload` | Plugin de auto-reload | `rec_control reload-zones` |
| **Maturidade** | 20+ anos | 10+ anos | 15+ anos |
| **Padrao** | Sim | Nao | Nao |

## Quando Usar Cada Um

### Unbound (padrao)

Melhor para a maioria das implantacoes. O Unbound oferece:

- Cache de referencia com otimizacao agressiva de NSEC
- Menor consumo de memoria
- Configuracao mais simples (arquivo unico)
- Testado em producao por mais de 20 anos

```yaml
# values.yaml
agent:
  engineType: unbound
```

### CoreDNS

Escolha CoreDNS quando:

- Sua equipe ja opera o CoreDNS e deseja uma unica tecnologia DNS
- Voce precisa de plugins personalizados (ex.: service discovery, middleware customizado)
- Voce deseja integracao de observabilidade nativa em Go

```yaml
# values.yaml
agent:
  engineType: coredns
  engineImages:
    coredns: coredns/coredns:1.12.1
```

### PowerDNS Recursor

Escolha PowerDNS quando:

- Voce precisa de scripting Lua para zonas de politica de resposta (RPZ)
- Sua organizacao padroniza no PowerDNS
- Voce precisa de suporte avancado a EDNS Client Subnet

```yaml
# values.yaml
agent:
  engineType: powerdns
  engineImages:
    powerdns: powerdns/pdns-recursor:5.2
```

## Imagens do Engine

Por padrao, o container do agent ja vem com o Unbound pre-instalado. Para outros engines, voce deve fornecer uma imagem personalizada ou especificar a imagem do engine:

```yaml
agent:
  engineImages:
    coredns: "my-registry/coredns:1.12.1"
    powerdns: "my-registry/pdns-recursor:5.2"
```

!!! warning
    A imagem padrao do agent inclui apenas os binarios do Unbound. Usar `engineType: coredns` ou `engineType: powerdns` sem fornecer a imagem do engine ou uma imagem personalizada do agent resultara em falha na inicializacao.
