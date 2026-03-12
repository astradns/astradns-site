# Contributing

## Repository Structure

AstraDNS is organized as a multi-repository project:

| Repository | Purpose |
|-----------|---------|
| `astradns-types` | Shared Go types: CRD definitions, engine interface, config contracts |
| `astradns-operator` | Control plane: Kubernetes operator, controllers, Helm chart |
| `astradns-agent` | Data plane: DNS proxy, metrics, logging, health checks, engine adapters |

Dependencies flow in one direction: `types` ← `operator` and `types` ← `agent`. The operator and agent never import each other.

## Development Setup

### Prerequisites

- Go 1.26.1+
- Docker
- kubectl
- Kind (for integration tests)
- Helm 3.12+

### Workspace Setup

```bash
git clone https://github.com/astradns/astradns-types.git
git clone https://github.com/astradns/astradns-operator.git
git clone https://github.com/astradns/astradns-agent.git

# Create go.work for local development
cat > go.work <<EOF
go 1.26.1

use (
    ./astradns-types
    ./astradns-operator
    ./astradns-agent
)
EOF
```

### Build and Test

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

### Run Integration Tests

```bash
cd astradns-operator
make test-integration
```

### Run Multi-K8s Version Matrix

```bash
cd astradns-operator
make test-integration-matrix
```

## Contribution Workflow

1. Fork the relevant repository
2. Create a feature branch: `git checkout -b feat/my-feature`
3. Make changes with tests
4. Run all tests: `make test`
5. Commit with [Conventional Commits](https://www.conventionalcommits.org/):
    - `feat:` new feature
    - `fix:` bug fix
    - `test:` test additions
    - `docs:` documentation
    - `refactor:` code restructure
    - `ci:` CI/CD changes
6. Open a Pull Request

## Code Review Checklist

- [ ] Tests pass (`make test`)
- [ ] Lint passes (`make lint`)
- [ ] No cross-repository imports (types ← agent, types ← operator only)
- [ ] Conventional commit messages
- [ ] CRD changes regenerated (`make manifests generate`)
- [ ] API compatibility maintained (no breaking changes in v1alpha1)
- [ ] No secrets or credentials in code

## Import Boundaries

```
astradns-types
    ↑           ↑
    |           |
astradns-operator   astradns-agent
```

- `astradns-operator` may import `astradns-types`
- `astradns-agent` may import `astradns-types`
- `astradns-operator` must NOT import `astradns-agent`
- `astradns-agent` must NOT import `astradns-operator`
- `astradns-types` must NOT import either
