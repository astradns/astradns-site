# Contribuciones

## Estructura del Repositorio

AstraDNS está organizado como un proyecto multi-repositorio:

| Repositorio | Propósito |
|-------------|-----------|
| `astradns-types` | Tipos compartidos de Go: definiciones de CRDs, interfaz del motor, contratos de configuración |
| `astradns-operator` | Plano de control: operador de Kubernetes, controladores, chart de Helm |
| `astradns-agent` | Plano de datos: proxy DNS, métricas, registro, verificaciones de salud, adaptadores de motor |

Las dependencias fluyen en una sola dirección: `types` <- `operator` y `types` <- `agent`. El operator y el agent nunca se importan entre sí.

## Configuración del Entorno de Desarrollo

### Prerrequisitos

- Go 1.26.1+
- Docker
- kubectl
- Kind (para pruebas de integración)
- Helm 3.12+

### Configuración del Workspace

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

### Como Ejecutamos Localmente

```bash
# 1) Contratos compartidos
cd astradns-types
make generate && make test && make vet

# 2) Plano de datos
cd astradns-agent
make test && make vet

# 3) Plano de control + chart Helm
cd astradns-operator
make test && make lint
```

### Como Probamos Antes del Merge

```bash
# Integracion full-stack (Kind + Helm + operator + agent)
cd astradns-operator
make test-integration
```

```bash
# Matriz de versiones Kubernetes usada en el release gate
cd astradns-operator
make test-integration-matrix
```

```bash
# Suite end-to-end
cd astradns-operator
make test-e2e
```

### Opcional: Ejecutar el Stack Manualmente en Kind

```bash
kind create cluster --name astradns-dev

helm upgrade --install astradns oci://ghcr.io/astradns/helm-charts/astradns \
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

## Flujo de Contribución

1. Haga un fork del repositorio correspondiente
2. Cree una rama de funcionalidad: `git checkout -b feat/my-feature`
3. Realice cambios con pruebas
4. Ejecute todas las pruebas: `make test`
5. Haga commit con [Conventional Commits](https://www.conventionalcommits.org/):
    - `feat:` nueva funcionalidad
    - `fix:` corrección de error
    - `test:` adiciones de pruebas
    - `docs:` documentación
    - `refactor:` reestructuración de código
    - `ci:` cambios de CI/CD
6. Abra un Pull Request

## Lista de Verificación para Code Review

- [ ] Las pruebas pasan (`make test`)
- [ ] El lint pasa (`make lint`)
- [ ] Sin importaciones entre repositorios (solo types <- agent, types <- operator)
- [ ] Mensajes de commit convencionales
- [ ] Cambios de CRDs regenerados (`make manifests generate`)
- [ ] Compatibilidad de API mantenida (sin cambios incompatibles en v1alpha1)
- [ ] Sin secretos ni credenciales en el código

## Límites de Importación

```
astradns-types
    ↑           ↑
    |           |
astradns-operator   astradns-agent
```

- `astradns-operator` puede importar `astradns-types`
- `astradns-agent` puede importar `astradns-types`
- `astradns-operator` NO debe importar `astradns-agent`
- `astradns-agent` NO debe importar `astradns-operator`
- `astradns-types` NO debe importar ninguno de los dos
