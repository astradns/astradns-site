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

### How We Run Locally

```bash
# 1) Shared contracts
cd astradns-types
make generate && make test && make vet

# 2) Data plane
cd astradns-agent
make test && make vet

# 3) Control plane + Helm chart
cd astradns-operator
make test && make lint
```

### How We Test Before Merge

```bash
# Full-stack integration (Kind + Helm + operator + agent)
cd astradns-operator
make test-integration
```

```bash
# Kubernetes version matrix used in release gate
cd astradns-operator
make test-integration-matrix
```

```bash
# End-to-end suite
cd astradns-operator
make test-e2e
```

### Optional: Run the Stack Manually in Kind

```bash
kind create cluster --name astradns-dev

helm upgrade --install astradns deploy/helm/astradns \
  --namespace astradns-system --create-namespace \
  --set agent.engineType=unbound \
  --set agent.network.mode=linkLocal \
  --set clusterDNS.forwardExternalToAstraDNS.enabled=true

kubectl apply -f - <<'EOF'
apiVersion: dns.astradns.com/v1alpha1
kind: DNSUpstreamPool
metadata:
  name: default
  namespace: astradns-system
spec:
  upstreams:
    - address: "1.1.1.1"
    - address: "8.8.8.8"
EOF
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
