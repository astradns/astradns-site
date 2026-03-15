# Variaveis de Ambiente

## Agent

| Variavel | Padrao | Descricao |
|----------|--------|-----------|
| `ASTRADNS_CONFIG_PATH` | `/etc/astradns/config/config.json` | Caminho para o arquivo de configuracao do engine (ou diretorio contendo `config.json`) |
| `ASTRADNS_ENGINE_TYPE` | `unbound` | Engine DNS a ser utilizado: `unbound`, `coredns`, `powerdns` ou `bind` |
| `ASTRADNS_ENGINE_CONFIG_DIR` | `/var/run/astradns/engine` | Diretorio onde o agent escreve os arquivos de configuracao especificos do engine |
| `ASTRADNS_LISTEN_ADDR` | `0.0.0.0:5353` | Endereco e porta para o listener do proxy DNS |
| `ASTRADNS_LOG_MODE` | `sampled` | Modo de logging de consultas: `full`, `sampled`, `errors-only` ou `off` |
| `ASTRADNS_LOG_SAMPLE_RATE` | `0.1` | Fracao de consultas bem-sucedidas a registrar (quando o modo e `sampled`). Intervalo: 0.0-1.0 |
| `ASTRADNS_HEALTH_ADDR` | `:8080` | Endereco do servidor HTTP de verificacao de saude |
| `ASTRADNS_METRICS_ADDR` | `:9153` | Endereco do servidor HTTP de metricas Prometheus |

## Operator

| Variavel | Padrao | Descricao |
|----------|--------|-----------|
| `ASTRADNS_ENGINE_TYPE` | `unbound` | Tipo de engine para renderizacao de config: `unbound`, `coredns`, `powerdns` ou `bind` |
| `ASTRADNS_AGENT_CONFIGMAP_NAME` | `astradns-agent-config` | Nome do ConfigMap escrito pelo operator |
| `POD_NAMESPACE` | *(da Downward API)* | Namespace onde o operator esta executando. Usado para determinar a localizacao do ConfigMap |
| `ASTRADNS_ENABLE_POOL_UNIQUENESS_WEBHOOK` | `false` | Habilitar o webhook de validacao que impoe um unico pool por namespace |

## Variaveis Gerenciadas pelo Helm

Estas variaveis sao definidas automaticamente ao implantar via Helm e geralmente nao devem ser sobrescritas manualmente:

| Variavel | Definida Por | Contexto |
|----------|-------------|----------|
| `ASTRADNS_ENGINE_TYPE` | `agent.engineType` | Operator e agent |
| `POD_NAMESPACE` | Downward API | Deployment do operator |
| `ASTRADNS_ENABLE_POOL_UNIQUENESS_WEBHOOK` | `webhook.enabled` | Deployment do operator |
