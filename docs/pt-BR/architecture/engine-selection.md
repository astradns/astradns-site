# Selecao de Engine

O AstraDNS suporta quatro engines DNS. A escolha e feita com um unico valor Helm: `agent.engineType`.

## Engines Suportados

| Engine | Uso tipico | Observacao |
|---|---|---|
| `unbound` | Padrao para a maioria dos clusters | Bom comportamento de cache recursivo e operacao simples |
| `coredns` | Times ja padronizados em CoreDNS | Consistencia operacional com o ecossistema CoreDNS |
| `powerdns` | Ambientes centrados em PowerDNS | Reaproveita praticas e tooling ja usados no ambiente |
| `bind` | Ambientes centrados em BIND | Bom encaixe para times com operacao consolidada em BIND |

## Politica de Imagens (Helm)

O chart gerencia repositorio e versao de imagem automaticamente:

- Imagem do operator: `ghcr.io/astradns/astradns-operator:v<appVersion>`
- Imagem do agent: `ghcr.io/astradns/astradns-agent:v<appVersion>-<engine>`

Assim, operator + agent + documentacao seguem a mesma linha de release. O usuario nao precisa escolher tags de imagem.

## Como Escolher o Engine

```yaml
agent:
  engineType: unbound # unbound | coredns | powerdns | bind
```

### Guia pratico

- Comece com `unbound`, salvo se voce ja tiver padrao organizacional claro para outro engine.
- Use `coredns`, `powerdns` ou `bind` quando seu time ja opera esse stack com playbooks e monitoracao consolidados.
- Mantenha um engine por ambiente; faca trocas em janela controlada.

## Validar o Engine Selecionado

```bash
# Engine propagado para o operator
kubectl -n astradns-system get deploy astradns-operator \
  -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="ASTRADNS_ENGINE_TYPE")].value}'

# Variante de imagem do agent selecionada pelo Helm
kubectl -n astradns-system get ds astradns-agent \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```
