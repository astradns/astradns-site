# Seleccion de Motor

AstraDNS soporta cuatro motores DNS. La seleccion se hace con un unico valor Helm: `agent.engineType`.

## Motores Soportados

| Motor | Uso tipico | Nota |
|---|---|---|
| `unbound` | Predeterminado para la mayoria de clusters | Buen comportamiento de cache recursivo y operacion simple |
| `coredns` | Equipos ya estandarizados en CoreDNS | Consistencia operativa con el ecosistema CoreDNS |
| `powerdns` | Entornos centrados en PowerDNS | Reutiliza practicas y tooling ya existentes |
| `bind` | Entornos centrados en BIND | Buen ajuste para equipos con operacion consolidada en BIND |

## Politica de Imagenes (Helm)

El chart administra repositorio y version de imagen automaticamente:

- Imagen del operator: `ghcr.io/astradns/astradns-operator:v<appVersion>`
- Imagen del agent: `ghcr.io/astradns/astradns-agent:v<appVersion>-<engine>`

Con esto, operator + agent + documentacion quedan alineados en la misma linea de release. El usuario no necesita elegir tags de imagen.

## Como Elegir el Motor

```yaml
agent:
  engineType: unbound # unbound | coredns | powerdns | bind
```

### Guia practica

- Empiece con `unbound`, salvo que ya exista un estandar organizacional claro para otro motor.
- Use `coredns`, `powerdns` o `bind` cuando su equipo ya opere ese stack con playbooks y monitoreo consolidados.
- Mantenga un motor por entorno; haga cambios en una ventana controlada.

## Validar el Motor Seleccionado

```bash
# Motor propagado al operator
kubectl -n astradns-system get deploy astradns-operator \
  -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="ASTRADNS_ENGINE_TYPE")].value}'

# Variante de imagen del agent seleccionada por Helm
kubectl -n astradns-system get ds astradns-agent \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```
