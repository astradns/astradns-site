# Contribuindo

## Estrutura do Repositorio

O AstraDNS e organizado como um projeto multi-repositorio:

| Repositorio | Proposito |
|-------------|-----------|
| `astradns-types` | Tipos Go compartilhados: definicoes de CRD, interface do engine, contratos de configuracao |
| `astradns-operator` | Plano de controle: operator Kubernetes, controllers, chart Helm |
| `astradns-agent` | Plano de dados: proxy DNS, metricas, logging, health checks, adaptadores de engine |

As dependencias fluem em uma unica direcao: `types` <- `operator` e `types` <- `agent`. O operator e o agent nunca importam um do outro.

## Configuracao de Desenvolvimento

### Pre-requisitos

- Go 1.26.1+
- Docker
- kubectl
- Kind (para testes de integracao)
- Helm 3.12+

### Configuracao do Workspace

```bash
git clone https://github.com/astradns/astradns-types.git
git clone https://github.com/astradns/astradns-operator.git
git clone https://github.com/astradns/astradns-agent.git

# Crie o go.work para desenvolvimento local
cat > go.work <<EOF
go 1.26.1

use (
    ./astradns-types
    ./astradns-operator
    ./astradns-agent
)
EOF
```

### Build e Testes

```bash
# Types
cd astradns-types
make generate && make test && make vet

# Agent
cd astradns-agent
make test && make vet

# Operator
cd astradns-operator
make manifests generate fmt vet test
```

### Executar Testes de Integracao

```bash
cd astradns-operator
make test-integration
```

### Executar Matriz de Versoes do Kubernetes

```bash
cd astradns-operator
make test-integration-matrix
```

## Fluxo de Contribuicao

1. Faca um fork do repositorio relevante
2. Crie um branch de feature: `git checkout -b feat/my-feature`
3. Faca as alteracoes com testes
4. Execute todos os testes: `make test`
5. Faca commit com [Conventional Commits](https://www.conventionalcommits.org/):
    - `feat:` novo recurso
    - `fix:` correcao de bug
    - `test:` adicao de testes
    - `docs:` documentacao
    - `refactor:` reestruturacao de codigo
    - `ci:` alteracoes de CI/CD
6. Abra um Pull Request

## Checklist de Code Review

- [ ] Testes passam (`make test`)
- [ ] Lint passa (`make lint`)
- [ ] Sem imports entre repositorios (apenas types <- agent, types <- operator)
- [ ] Mensagens de commit convencionais
- [ ] Alteracoes de CRD regeneradas (`make manifests generate`)
- [ ] Compatibilidade de API mantida (sem alteracoes incompativeis em v1alpha1)
- [ ] Sem segredos ou credenciais no codigo

## Limites de Importacao

```
astradns-types
    ^           ^
    |           |
astradns-operator   astradns-agent
```

- `astradns-operator` pode importar `astradns-types`
- `astradns-agent` pode importar `astradns-types`
- `astradns-operator` NAO deve importar `astradns-agent`
- `astradns-agent` NAO deve importar `astradns-operator`
- `astradns-types` NAO deve importar nenhum dos dois
