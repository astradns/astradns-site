# Validación de SLOs

AstraDNS define cinco Objetivos de Nivel de Servicio que se validan antes de cada lanzamiento.

## Objetivos de SLO

| # | SLO | Objetivo | Cómo Medir |
|---|-----|----------|------------|
| 1 | Tiempo de instalación | < 5 minutos | Tiempo desde `helm install` hasta que todos los pods estén Running |
| 2 | Latencia DNS p95 | <= línea base sin AstraDNS | `histogram_quantile(0.95, rate(astradns_upstream_latency_seconds_bucket[5m]))` |
| 3 | Ratio de aciertos de caché | > 30% después del calentamiento | `rate(cache_hits) / (rate(cache_hits) + rate(cache_misses))` |
| 4 | Tiempo de recuperación | < 30 segundos | Tiempo desde el reinicio del pod del agent hasta Ready |
| 5 | Fallas inducidas por AstraDNS | 0 | Ninguna respuesta SERVFAIL causada por AstraDNS en sí |

## Ejecución de la Validación de SLOs

### Automatizada

```bash
make test-slo
```

Esto ejecuta el script `test/slo/validate-mvp.sh` que:

1. Verifica la disponibilidad de los pods
2. Mide la línea base de latencia de consultas DNS
3. Ejecuta consultas repetidas para construir el caché
4. Mide el ratio de aciertos de caché
5. Simula el reinicio de un pod y mide el tiempo de recuperación
6. Reporta aprobado/fallido para cada SLO

### Configuración

| Variable | Por Defecto | Descripción |
|----------|-------------|-------------|
| `SLO_NAMESPACE` | `astradns-system` | Namespace a probar |
| `SLO_RELEASE` | `astradns` | Nombre del release de Helm |
| `SLO_ITERATIONS` | `100` | Número de consultas DNS por prueba |
| `SLO_DOMAIN` | `example.com` | Dominio a consultar |

### Validación Manual

#### SLO 1: Tiempo de Instalación

```bash
time helm install astradns deploy/helm/astradns \
  --namespace astradns-system --create-namespace

kubectl wait --for=condition=Ready pods \
  -l app.kubernetes.io/part-of=astradns \
  -n astradns-system --timeout=300s
```

#### SLO 2: Latencia

```bash
# Baseline (without AstraDNS)
kubectl run dns-baseline --rm -it --restart=Never \
  --image=busybox:1.37 -- sh -c \
  'for i in $(seq 1 100); do nslookup example.com > /dev/null 2>&1; done'

# With AstraDNS (compare times)
```

#### SLO 3: Ratio de Aciertos de Caché

```bash
# After warm-up queries
curl -s http://localhost:9153/metrics | \
  grep -E 'astradns_cache_(hits|misses)_total'
```

#### SLO 4: Tiempo de Recuperación

```bash
# Delete an agent pod and measure recovery
kubectl delete pod -n astradns-system -l app.kubernetes.io/component=agent --wait=false
time kubectl wait --for=condition=Ready pods \
  -l app.kubernetes.io/component=agent \
  -n astradns-system --timeout=60s
```
