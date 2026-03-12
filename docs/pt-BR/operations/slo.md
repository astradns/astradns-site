# Validacao de SLO

O AstraDNS define cinco Objetivos de Nivel de Servico que sao validados antes de cada release.

## Metas de SLO

| # | SLO | Meta | Como Medir |
|---|-----|------|------------|
| 1 | Tempo de instalacao | < 5 minutos | Tempo desde `helm install` ate todos os pods Running |
| 2 | Latencia DNS p95 | <= baseline sem AstraDNS | `histogram_quantile(0.95, rate(astradns_upstream_latency_seconds_bucket[5m]))` |
| 3 | Taxa de acerto de cache | > 30% apos aquecimento | `rate(cache_hits) / (rate(cache_hits) + rate(cache_misses))` |
| 4 | Tempo de recuperacao | < 30 segundos | Tempo desde o reinicio do pod do agent ate Ready |
| 5 | Falhas induzidas pelo AstraDNS | 0 | Nenhuma resposta SERVFAIL causada pelo proprio AstraDNS |

## Executando a Validacao de SLO

### Automatizada

```bash
make test-slo
```

Isso executa o script `test/slo/validate-mvp.sh` que:

1. Verifica a prontidao dos pods
2. Mede a latencia baseline de consulta DNS
3. Executa consultas repetidas para popular o cache
4. Mede a taxa de acerto de cache
5. Simula reinicio do pod e mede o tempo de recuperacao
6. Reporta aprovacao/reprovacao para cada SLO

### Configuracao

| Variavel | Padrao | Descricao |
|----------|--------|-----------|
| `SLO_NAMESPACE` | `astradns-system` | Namespace para testes |
| `SLO_RELEASE` | `astradns` | Nome do release Helm |
| `SLO_ITERATIONS` | `100` | Numero de consultas DNS por teste |
| `SLO_DOMAIN` | `example.com` | Dominio a ser consultado |

### Validacao Manual

#### SLO 1: Tempo de Instalacao

```bash
time helm install astradns deploy/helm/astradns \
  --namespace astradns-system --create-namespace

kubectl wait --for=condition=Ready pods \
  -l app.kubernetes.io/part-of=astradns \
  -n astradns-system --timeout=300s
```

#### SLO 2: Latencia

```bash
# Baseline (sem AstraDNS)
kubectl run dns-baseline --rm -it --restart=Never \
  --image=busybox:1.37 -- sh -c \
  'for i in $(seq 1 100); do nslookup example.com > /dev/null 2>&1; done'

# Com AstraDNS (compare os tempos)
```

#### SLO 3: Taxa de Acerto de Cache

```bash
# Apos consultas de aquecimento
curl -s http://localhost:9153/metrics | \
  grep -E 'astradns_cache_(hits|misses)_total'
```

#### SLO 4: Tempo de Recuperacao

```bash
# Exclua um pod do agent e meça a recuperacao
kubectl delete pod -n astradns-system -l app.kubernetes.io/component=agent --wait=false
time kubectl wait --for=condition=Ready pods \
  -l app.kubernetes.io/component=agent \
  -n astradns-system --timeout=60s
```
